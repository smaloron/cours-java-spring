# Thymeleaf

Thymeleaf est un moteur de template Java qui permet de produire du contenu HTML ou XML dynamique.
Avec Spring Boot, son intégration est très simple, car toute la configuration est prise en charge par le framework.

**Avantage de Thymeleaf**

- Syntaxe naturelle (un modèle Thymeleaf peut être lu par n'importe quel éditeur HTML).
- Lecture et apprentissage facile.
- Support de SpEL (Spring Expression Language).
- Intégration avec les objets de modèle Spring (Model, ModelMap ...).
- Possibilité de factoriser les vues avec des fragments et des gabarits (layouts).
- Contrairement à JSP ou à JSTL, Thymeleaf ne permet pas d'intégrer de la logique métier aux vues.

**Installation**

Pour utiliser Thymeleaf, il suffit d'ajouter la dépendance à `pom.xml`. Spring Boot se charge alors de la
configuration. Par défaut le dossier dans lequel il recherchera les modèles est `src/main/resources/templates`.

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

## Utilisation de Thymeleaf

Thymeleaf ne modifie pas la structure HTML (ou XML), il se contente d'ajouter des attributs aux balises.
Cette approche permet de facilement dynamiser une maquette statique et rend le code des vues totalement compatible
avec n'importe quel éditeur HTML.

### Un exemple de vue Thymeleaf

```HTML
<!DOCTYPE html>
<html xmlns:th="http://www.Thymeleaf.org">
<head>
    <title>Hello Page</title>
    <meta charset="UTF-8"/>
</head>
<body>
<h1 th:text="${message}">Message par défaut</h1>
</body>
</html>
```

- `<html xmlns:th="http://www.Thymeleaf.org">` : Importe le namespace Thymeleaf dans le document
- `th:text="${message}"` : Affiche l'attribut message transmis à la vue par le contrôleur.

**Le code du contrôleur**

Dans le contrôleur, il faut injecter une instance de `org.springframework.ui.Model`
et lui ajouter des attributs (`addAttribute(key, value)`) sous la forme d'une paire clef/valeur.
Les clefs permettront au template Thymeleaf d'accéder aux valeurs.
Les valeurs ne sont pas limitées à des variables scalaires, il est tout à fait possible de passer des objets.

```java
package fr.formation.spring.demo.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

// Indique que ce contrôleur sert des vues (pas REST)
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String hello(Model model) {
        // 💬 Ajoute une variable à la vue
        model.addAttribute("message", "Bonjour Spring !");
        // 🔁 Retourne le nom de la vue (hello.html)
        return "hello";
    }
}
```

### Variante HTML5

Il existe une syntaxe alternative qui évite d'avoir à déclarer le namespace.
Dans ce cas, il faut remplacer l'attribut `th:instruction` par `data-th-instruction`.
Ainsi `th:text` devient `data-th-text`.
Les puristes de la spécification HTML 5 préfèreront cette alternative
puisque les namespace ont été abandonnés dans cette nouvelle mouture.

**La vue précédente avec cette syntaxe**

```HTML
<!DOCTYPE html>
<html>
<head>
    <title>Hello Page</title>
    <meta charset="UTF-8"/>
</head>
<body>
<h1 data-th-text="${message}">Message par défaut</h1>
</body>
</html>
```

## Les expressions

Le contenu d'un attribut Thymeleaf est souvent une expression (un contenu dynamique),
mais il en existe plusieurs sortes.

- `${...}` : pour les expressions à évaluer
- `*{...}` : pour les expressions de sélection qui permettent de sélectionner une propriété d'un objet lié. Ceci est
  très utile pour les formulaires.
- `#{...}` : pour les expressions de message. Les valeurs sont extraites d'un fichier de ressources (`.propreties`),
  cela permet de centraliser et factoriser les messages.
- `@{...}` : pour les expressions d'URL. Le contenu est résolu pour générer un lien en fonction du contexte de
  l'application.

Les expressions peuvent inclure des variables, des données littérales, des opérateurs
et même des appels de méthodes si l'on a ajouté un objet dans le `model` du template.

Cette dernière option est toutefois à utiliser avec parcimonie.
Il faut en effet se garder d'injecter de la logique métier dans les templates.

### Échappement des caractères

L'échappement des caractères est automatique avec Thymeleaf.
Les caractères spéciaux sont convertis en entité HTML ce qui protège contre les attaques XSS.

**Exemple**

```java

@GetMapping("/escape-test")
public String test(Model model) {
    model.addAttribute("message", "<b>Hello</b> & welcome!");
    return "escape-test";
}
```

```html
<p th:text="${message}"></p>
```

**Résultat**

```
<p>&lt;b&gt;Hello&lt;/b&gt; &amp; welcome!</p>
```

Il est possible de forcer l'affichage des caractères sans échappement
en ajoutant un `u` (pour unescaped) devant l'attribut `text`.

```html
<p th:utext="${message}"></p>
```

### Les objets implicites

Un template Thymeleaf a automatiquement accès aux objets suivants :

- La session (variable `session`).
- Le contexte de l'application (variable `application`).
- Les paramètres (variable `param`).

#### Exemple d'accès aux objets implicites

```java

@GetMapping(value = "/scopes")
public String ScopeTest(
        @RequestParam String name,
        HttpSession session,
        HttpServletRequest request
) {
    // scope request : durée, une seule requête
    request.setAttribute("message", "Il fait beau");

    // scope session : durée, la session utilisateur
    session.setAttribute("favoriteColor", "rouge");

    // scope application : durée, la vie du serveur
    ServletContext context = request.getServletContext();
    context.setAttribute("appName", "Spring Framework");

    return "thymeleaf-test/implicit-objects";
}
```

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Objets implicites</title>
</head>
<body>

<h1>Objets implicites</h1>

<h2>Accès aux paramètres (querystring)</h2>
<p th:text="'Hello, ' + ${param.name} + '!'"></p>

<h2>Accès à la session</h2>
<p th:text="'vous aimez le, ' + ${session.favoriteColor} + '!'"></p>

<h2>Accès au contexte de l'application</h2>
<p th:text="${application.appName}"></p>

</body>
</html>
```

### Les expressions d'URL

Les expressions d'URL permettent de créer des liens en prenant en compte le contexte d'exécution du serveur et en
ajoutant d'éventuels paramètres.

**Syntaxe de base**

```html
<a th:href="@{/page}">Lien simple</a>
```

#### Lien avec une variable dans le path

```html
<a th:href="@{/product/{id}(id=${productId})}">Voir produit</a>
```

La valeur du ou des paramètres est passée entre les parenthèses.

#### Lien avec une variable dans le query string

```html
<a th:href="@{/search(query=${searchQuery})}">Rechercher</a>
```

Très similaire au précédent, mais sans les accolades.

#### Combinaison

Il est possible de combiner les deux types de variables dans un même URL.

```html
<a th:href="@{/user/{id}/profile(id=${userId}, active=true)}">Profil</a>
```

**Résultat**

```html
<a href="/user/12/profile?active=true">Profil</a>
```

#### Resources statiques

Ce principe de création d'URL fonctionne également avec les images, les liens CSS ou Javascript.
Selon les cas, il faudra utiliser les attributs `th:href` ou `th:src`.

### Les expressions de sélection

Là où une expression classique `${}` recherche une variable dans le `Model` d'un template `Thymeleaf`,
l'expression de sélection limite cette portée à un seul objet déclaré dans une balise parente.
Cette expression permet de simplifier la syntaxe en factorisant l'objet source.

**Exemple**
En imaginant un objet `User` possédant les propriétés `id` et `name`.

```html

<div th:object="${user}">
    <p th:text="*{id}"></p>
    <p th:text="*{name}"></p>
</div>
```

#### Dans les itérations

```html

<tr th:each="user : ${users}" th:object="${user}">
    <td th:text="*{id}"></td>
    <td th:text="*{name}"></td>
</tr>
```

- Chaque ligne `<tr>` a son propre contexte user.

- Les `*{}` accèdent directement aux propriétés du `user` courant.

#### Dans un formulaire

```html

<form th:object="${user}">
    <label>Nom :</label>
    <input type="hidden" th:field="*{id}"/>

    <label>Email :</label>
    <input type="text" th:field="*{name}"/>
</form>
```

Avec `th:field`, `Thymeleaf` génère automatiquement :

- l'attribut `name`

- l'attribut `id`

- l'attribut `value`

Il gère aussi la récupération automatique en cas d'erreur de validation (Spring MVC BindingResult).

### Les expressions de message

Les expressions de message sont calquées sur le modèle de l'internationalisation de `JSTL`.
Ce sont les mêmes fichiers `.properties` qui sont utilisés. Ces fichiers

**Un fichier de message**

`messages.properties` dans `src/main/resources/`

```
page.title=Bienvenue
user.name=Nom
user.email=Email
```

Pour gérer différentes langues, il faut ajouter le code de la langue en suffixe,
`messages_en.properties` pour l'anglais par exemple.

**Utilisation des messages dans Thymeleaf**

```html
<h1 th:text="#{page.title}"></h1>

<label th:text="#{user.name}"></label>
<input type="text" name="name"/>

<label th:text="#{user.email}"></label>
<input type="email" name="email"/>
```

#### Message paramétré

```
greeting=Bonjour {0} !
order.summary=Commande {0} pour {1} articles

```

```html
<!-- passage d'un paramètre -->
<p th:text="#{greeting(${userName})}"></p>

<!-- passage de deux paramètres -->
<p th:text="#{order.summary(${orderId}, ${itemCount})}"></p>
```

#### Valeur par défaut

Cette valeur sera utilisée si la clef demandée est absente du fichier `.properties`.

```html
<span th:text="#{inconnu.message(default='Message par défaut')}"></span>
```

#### Configuration

La configuration des messages s'effectue dans le fichier `application.properties` situé dans `src/main/resources`.
Il est possible de définir le nom de base du fichier des messages et la locale par défaut.

```
spring.messages.basename=messages
spring.mvc.locale=fr
```

#### Internationalisation

Pour changer de langue, il existe deux solutions :

- Passer la locale dans le query string
- Utiliser la locale du navigateur transmise dans l'en-tête HTTP

Dans les deux cas, il faut créer une classe de configuration
qui sera automatiquement scannée par Spring.

##### Utilisation de la locale du navigateur

```java
package fr.formation.spring.demo.configuration;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver;

import java.util.Locale;

@Configuration
public class WebConfig {

    @Bean
    public LocaleResolver localeResolver() {
        AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
        // Définir la locale par défaut
        resolver.setDefaultLocale(Locale.FRENCH);
        return resolver;
    }
}
```

##### Passage de la locale das le query string

```java
package fr.formation.spring.demo.configuration;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;

import java.util.Locale;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver slr = new SessionLocaleResolver();
        slr.setDefaultLocale(Locale.FRENCH);
        return slr;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
        lci.setParamName("lang"); // paramètre d'URL: ?lang=en
        return lci;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
```

## Les attributs HTML de Thymeleaf

### Les conditions

Thymeleaf propose deux structures conditionnelles, `th:if` et `th:unless`. Ces attributs sont placés dans une balise
et admettent une expression booléenne dont l'évaluation conditionnera l'affichage de la balise.

#### Exemples

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Test de l'age</title>
</head>
<body>

<p th:if="${age} < 18">
    Vous êtes mineur
</p>

<p th:if="${age} >= 18">
    Vous êtes majeur
</p>

</body>
</html>
```

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Test du de la maison de sorcier</title>
</head>
<body>
<div th:with="startsWithAnS = ${name.charAt(0)} == 'S'">
    <p th:unless="${startsWithAnS}">
        Vous êtes d'une autre maison
    </p>

    <p th:if="${startsWithAnS}">
        Vous êtes Slitherin
    </p>
</div>
</body>
</html>
```

Dans ce dernier exemple, l'attribut `th:with` définit une variable dont la portée est celle de la balise hôte.
Cela permet de factoriser le test et de simplifier la lecture du code.

**Petits exercices**

- Écrire les contrôleurs qui affichent ces vues.
- Dans le premier exemple, factoriser l'expression booléenne avec `th:with`.

#### Le Switch

Pour tester plusieurs valeurs d'une même variable, `Thymeleaf` dispose d'une structure `switch`.

```html

<div th:switch="${user.role}">
    <p th:case="'ADMIN'">Welcome admin!</p>
    <p th:case="'USER'">Welcome user!</p>
    <p th:case="*">Welcome guest!</p> <!-- * = par défaut -->
</div>
```

### Les itérations

Les boucles Thymeleaf sont de type foreach. La balise qui héberge la boucle sera dupliquée à chaque itération.

La syntaxe est la suivante :

```
th:each="variable-d-iteration : ${collection-source}"
```

#### Un exemple

```java
    // url : /boucle1?tags=Java,Python,Erlang
@GetMapping(value = "/boucle1", params = "tags")
public String LoopWithTags(
        @RequestParam List<String> tags, Model model
) {
    model.addAttribute("tagList", tags);
    return "thymeleaf-test/tags";
}
```

```html
<!-- thymeleaf-test/tags.html -->
<!DOCTYPE html>
<html lang="fr" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Liste des tags</title>
</head>
<body>

<ul>
    <li th:each="tag : ${tagList}" th:text="${tag}"></li>
</ul>

</body>
</html>
```

#### Autre exemple

```java

@GetMapping(value = "/boucle2")
public String LoopWithEdibles(Model model) {
    List<EdibleDTO> food = new ArrayList<>();
    food.add(new EdibleDTO("Pomme", "fruit"));
    food.add(new EdibleDTO("Poire", "fruit"));
    food.add(new EdibleDTO("Panais", "légume"));
    food.add(new EdibleDTO("Potiron", "légume"));

    model.addAttribute("foodList", food);

    return "thymeleaf-test/edibles";
}
```

```html
<!DOCTYPE html>
<html lang="fr" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Liste des ingrédients</title>
</head>
<body>

<table>
    <tr th:each="food : ${foodList}">
        <td th:text="${food.name}"></td>
        <td th:text="${food.category}"></td>
    </tr>
</table>

</body>
</html>
```

#### Accès à l'état de l'itération

Dans une itération, il est possible de définir une deuxième variable
chargée de stocker l'état de l'itération.

```html

<tr th:each="user, stat : ${users}">
    <td th:text="${stat.index}"></td> <!-- Index de départ 0 -->
    <td th:text="${stat.count}"></td> <!-- Compte (1, 2, 3...) -->
    <td th:text="${stat.size}"></td> <!-- Taille totale -->
    <td th:text="${stat.even}"></td> <!-- true si ligne paire -->
    <td th:text="${stat.odd}"></td>  <!-- true si ligne impaire -->
    <td th:text="${user.name}"></td>
</tr>
```

- `stat.index` → position (0, 1, 2, …)

- `stat.count` → position (1, 2, 3, …)

- `stat.size` → taille totale de la liste

- `stat.even` / `stat.odd` → parité

- `stat.first` → true pour le premier élément

- `stat.last` → true pour le dernier élément

#### Itération sur une Map

Thymeleaf permet aussi de boucler sur une Map.

**Exemple :**

```html

<ul>
    <li th:each="entry : ${myMap}">
        <span th:text="${entry.key}">Key</span> :
        <span th:text="${entry.value}">Value</span>
    </li>
</ul>
```

En Java, myMap est un `Map<String, String>` par exemple.

### Récapitulatif de tous les attributs

| Attribut                                                     | Description                                                                             | Exemple / Usage Typique                                                       |
|--------------------------------------------------------------|-----------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| **Affichage Texte**                                          |                                                                                         |                                                                               |
| `th:text`                                                    | Affiche du texte (échappe les caractères HTML pour la sécurité).                        | `<p th:text="${message}">Texte par défaut</p>`                                |
| `th:utext`                                                   | Affiche du texte sans échapper le HTML (utiliser avec prudence).                        | `<div th:utext="${htmlContent}">Contenu HTML</div>`                           |
| `th:value`                                                   | Définit l'attribut `value` (inputs, options, etc.).                                     | `<input type="text" th:value="${user.name}">`                                 |
| **Conditions**                                               |                                                                                         |                                                                               |
| `th:if`                                                      | Affiche l'élément si la condition est vraie.                                            | `<div th:if="${isAdmin}">Section Admin</div>`                                 |
| `th:unless`                                                  | Affiche l'élément si la condition est fausse (inverse de `th:if`).                      | `<p th:unless="${cart.isEmpty()}">Votre panier n'est pas vide.</p>`           |
| `th:switch`, `th:case`                                       | Structure conditionnelle multiple (similaire au switch/case).                           | `<div th:switch="${user.role}"><p th:case="'ADMIN'">...</p></div>`            |
| **Itérations**                                               |                                                                                         |                                                                               |
| `th:each`                                                    | Itère sur une collection (List, Set, Map, Array...) et répète l'élément.                | `<ul><li th:each="item : ${items}" th:text="${item}"></li></ul>`              |
| **Attributs HTML**                                           |                                                                                         |                                                                               |
| `th:attr`                                                    | Définit un ou plusieurs attributs HTML dynamiquement.                                   | `<img th:attr="alt=${altText}, title=${titleText}">`                          |
| `th:attrappend`                                              | Ajoute une valeur à la fin d'un attribut existant (ex: `class`, `style`).               | `<div class="base" th:attrappend="class=' ' + ${dynamicClass}">`              |
| `th:attrprepend`                                             | Ajoute une valeur au début d'un attribut existant.                                      | `<input style="margin: auto;" th:attrprepend="style='color: red; '">`         |
| `th:classappend`                                             | Raccourci pratique pour ajouter une classe CSS.                                         | `<div class="item" th:classappend="${isError} ? 'error'">`                    |
| `th:checked`, <br />`th:selected`, <br />`th:disabled`, etc. | Gère les attributs HTML booléens (définit si l'expression est vraie).                   | `<input type="checkbox" th:checked="${user.isActive}">`                       |
| **URLs et Liens**                                            |                                                                                         |                                                                               |
| `th:href`, `th:src`, `th:action`...                          | Définit des attributs d'URL en utilisant la syntaxe `@{...}`.                           | `<a th:href="@{/users/{id}(id=${user.id})}">Profil</a>`                       |
| **Fragments (Layouts)**                                      |                                                                                         |                                                                               |
| `th:fragment`                                                | Définit une portion de template réutilisable (un fragment).                             | `<footer th:fragment="footer">...</footer>` (dans `fragments.html`)           |
| `th:insert`                                                  | Insère un fragment *à l'intérieur* de la balise hôte.                                   | `<div th:insert="~{fragments :: footer}"></div>`                              |
| `th:replace`                                                 | Remplace la balise hôte *par* le fragment.                                              | `<footer th:replace="~{fragments :: footer}"></footer>`                       |
| `th:include`                                                 | **Déprécié** (préférer `th:insert`). Insère le *contenu* du fragment.                   | `<div th:include="~{fragments :: content}"></div>`                            |
| **Variables / Objets**                                       |                                                                                         |                                                                               |
| `th:with`                                                    | Définit une ou plusieurs variables locales dans la portée de l'élément.                 | `<div th:with="isEven=${iterStat.even}">...</div>`                            |
| `th:object`                                                  | Sélectionne un objet du contexte pour utiliser `*{...}` dans les enfants (formulaires). | `<form th:object="${user}" th:action="@{/save}">...</form>`                   |
| **Autres**                                                   |                                                                                         |                                                                               |
| `th:remove`                                                  | Contrôle la suppression de la balise ou de son contenu (`all`, `body`, `tag`...).       | `<p th:remove="tag">Contenu seul</p>`                                         |
| `th:block`                                                   | Balise "virtuelle" non rendue, utile pour appliquer de la logique (ex: `th:each`).      | `<th:block th:each="msg : ${messages}"><p th:text="${msg}"></p></th:block>`   |
| `th:inline`                                                  | Active le traitement inline dans JS (`javascript`) ou CSS (`css`).                      | `<script th:inline="javascript">var name = /*[[${user.name}]]*/ '';</script>` |

## Les objets utilitaires de Thymeleaf

Thymeleaf fournit un ensemble riche **d'objets utilitaires (Expression Utility Objects)**. 
Ces objets, accessibles directement dans les expressions Thymeleaf via la syntaxe `#nomObjet`, 
offrent des méthodes pratiques pour effectuer des opérations courantes directement dans le template, 
rendant le code plus lisible et en évitant de surcharger le contrôleur Java avec de la logique de présentation.

### Accès aux Objets Utilitaires

Les objets utilitaires sont disponibles par défaut dans le dialecte standard de Thymeleaf (`Standard Dialect`), 
qui est automatiquement configuré par Spring Boot lors de l'utilisation du starter `spring-boot-starter-thymeleaf`. 
Ils sont accessibles en utilisant le préfixe `#` suivi du nom de l'objet (par exemple, `#strings`, `#numbers`, `#dates`).

### Les Utilitaires Essentiels

#### Utilitaires pour les Chaînes de Caractères `#strings`

L'objet `#strings` fournit des méthodes pour la manipulation des chaînes de caractères. 
Il est particulièrement utile pour les vérifications, les transformations et les formatages simples.

*   **Fonctions courantes :**
  *   `isEmpty(String)`: Vérifie si une chaîne est vide ou nulle.
  *   `contains(String, String)`: Vérifie si une chaîne en contient une autre.
  *   `startsWith(String, String)`, `endsWith(String, String)`: Vérifications de préfixe/suffixe.
  *   `toUpperCase(String)`, `toLowerCase(String)`: Changement de casse.
  *   `substring(String, int, int)`, `substringAfter(String, String)`, `substringBefore(String, String)`: Extraction de sous-chaînes.
  *   `trim(String)`: Suppression des espaces en début et fin.
  *   `length(String)`: Obtention de la longueur.
  *   `equals(String, String)`, `equalsIgnoreCase(String, String)`: Comparaison.

*   **Exemple :**
    Supposons qu'un contrôleur ajoute un attribut `description` au modèle :
    ```java
    // Dans le contrôleur Spring MVC
    @Controller
    public class ProductController {
        @GetMapping("/product/details")
        public String showProductDetails(Model model) {
            String description = "  Un produit exceptionnel! ";
            model.addAttribute("productDescription", description);
            model.addAttribute("productCode", "PROD-123");
            return "product-details";
        }
    }
    ```

    Dans le template `product-details.html`:
    ```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head><title>Détails Produit</title></head>
    <body>
        <h1>Description</h1>
        <!-- Affiche la description en majuscules et sans espaces superflus -->
        <p th:text="${#strings.toUpperCase(#strings.trim(productDescription))}">
            Description par défaut
        </p>

        <!-- Affiche le code seulement s'il commence par "PROD-" -->
        <div th:if="${#strings.startsWith(productCode, 'PROD-')}">
            <p>Code Produit Valide: <span th:text="${productCode}"></span></p>
        </div>

        <!-- Vérifie si la description est vide -->
        <p th:if="${#strings.isEmpty(productDescription)}">
            Aucune description fournie.
        </p>
    </body>
    </html>
    ```

#### Utilitaires pour les Nombres `#numbers`

L'objet `#numbers` aide à formater les nombres selon des motifs spécifiques, en tenant compte de la localisation (locale). 
C'est essentiel pour afficher des devises, des pourcentages ou des décimaux formatés.

*   **Fonctions courantes :**
  *   `formatDecimal(Number, int, int)`: Formate un nombre avec un nombre minimum de chiffres entiers et un nombre fixe de décimales.
  *   `formatDecimal(Number, int, String, int, String)`: Version plus complète avec spécification des séparateurs.
  *   `formatCurrency(Number)`: Formate un nombre comme une devise, en utilisant la locale courante.
  *   `formatPercent(Number, int, int)`: Formate un nombre comme un pourcentage.

*   **Exemple :**
    ```java
    // Dans le contrôleur
    @Controller
    public class OrderController {
        @GetMapping("/order/summary")
        public String showOrderSummary(Model model) {
            model.addAttribute("totalPrice", 1250.75);
            model.addAttribute("taxRate", 0.20);
            return "order-summary";
        }
    }
    ```

    Dans le template `order-summary.html`:
    ```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org" lang="fr"> <!-- Important pour la locale -->
    <head><title>Résumé Commande</title></head>
    <body>
        <h1>Résumé de la commande</h1>
        <!-- Affiche le prix formaté en devise (Euro pour la locale 'fr') -->
        <p>Total : 
            <span th:text="${#numbers.formatCurrency(totalPrice)}">
                0.00 €
            </span>
        </p>

        <!-- Affiche le taux de taxe en pourcentage avec 1 décimale -->
        <p>Taxe : 
            <span th:text="${#numbers.formatPercent(taxRate, 1, 1)}">
                0%
            </span>
        </p>

        <!-- Affiche le prix avec formatage spécifique -->
        <p>Prix (format spécifique) : 
            <span th:text="${#numbers.formatDecimal(totalPrice, 3, 2, 'COMMA')}">
                0,00
            </span>
        </p>
    </body>
    </html>
    ```
    **Note:** Le formatage dépend de la locale configurée pour la requête. Spring Boot tente de la déterminer automatiquement (via l'en-tête `Accept-Language`, un `LocaleResolver`, etc.).

#### Utilitaires pour les Dates et Heures `#dates`, `#calendars`, `#temporals`

Thymeleaf propose plusieurs objets pour manipuler les dates et heures :
*   `#dates`: Pour les objets `java.util.Date`.
*   `#calendars`: Pour les objets `java.util.Calendar`.
*   `#temporals`: **(Recommandé)** Pour les objets de l'API Java 8+ Date/Time (`java.time.*` comme `LocalDate`, `LocalDateTime`, `ZonedDateTime`).

*   **Fonctions courantes (`#temporals` est préféré pour Java 8+) :**
  *   `format(Temporal, String)`: Formate une date/heure selon un motif (ex: 'dd/MM/yyyy HH:mm').
  *   `formatISO(Temporal)`: Formate selon la norme ISO 8601.
  *   `day(Temporal)`, `month(Temporal)`, `year(Temporal)`, `hour(Temporal)`, etc.: Extrait des composants spécifiques.
  *   `dayOfWeekName(Temporal)`, `monthName(Temporal)`: Obtient le nom du jour/mois (localisé).
  *   `createNow()`: Obtient la date/heure actuelle.

*   **Exemple :**
    ```java
    // Dans le contrôleur
    @Controller
    public class EventController {
        @GetMapping("/event/info")
        public String showEventInfo(Model model) {
            model.addAttribute("eventDate", java.time.LocalDate.of(2024, 12, 25));
            model.addAttribute("eventTime", java.time.LocalDateTime.now());
            return "event-info";
        }
    }
    ```

    Dans le template `event-info.html`:
    ```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org" lang="fr">
    <head><title>Info Événement</title></head>
    <body>
        <h1>Informations sur l'événement</h1>
        <!-- Formater une LocalDate avec #temporals -->
        <p>Date : 
            <span th:text="${#temporals.format(eventDate, 'dd MMMM yyyy')}">
                Date
            </span>
        </p>

        <!-- Formater une LocalDateTime avec #temporals -->
        <p>Heure de publication : 
            <span th:text="${#temporals.format(eventTime, 'HH:mm:ss')}">
                Heure
            </span>
        </p>

        <!-- Extraire le jour de la semaine -->
        <p>Jour : 
            <span th:text="${#temporals.dayOfWeekName(eventDate)}">
                Jour
            </span
        ></p>

        <!-- Utilisation de #dates si on avait une java.util.Date -->
        <!-- <p th:if="${legacyDate != null}" th:text="${#dates.format(legacyDate, 'dd-MM-yyyy')}"></p> -->
    </body>
    </html>
    ```

#### Utilitaires pour les Collections et Tableaux (`#lists`, `#arrays`, `#sets`, `#maps`)

Ces utilitaires permettent d'effectuer des opérations de base sur les collections Java.

*   **Fonctions courantes :**
  *   `size(Collection/Array/Map)`: Retourne la taille.
  *   `isEmpty(Collection/Array/Map)`: Vérifie si la collection est vide ou nulle.
  *   `contains(Collection/Array, Object)`: Vérifie si un élément est présent.
  *   `containsAll(Collection/Array, Collection/Array)`: Vérifie si tous les éléments sont présents.
  *   Pour `#maps`: `containsKey(Map, Object)`, `containsValue(Map, Object)`.

*   **Exemple :**
    ```java
    // Dans le contrôleur
    @Controller
    public class DataController {
        @GetMapping("/data/view")
        public String showData(Model model) {
            java.util.List<String> items = java.util.Arrays.asList("Pomme", "Banane", "Orange");
            java.util.Map<String, Boolean> features = java.util.Map.of("featureA", true, "featureB", false);
            model.addAttribute("itemList", items);
            model.addAttribute("featureMap", features);
            return "data-view";
        }
    }
    ```

    Dans le template `data-view.html`:
    ```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head><title>Visualisation Données</title></head>
    <body>
        <h1>Données</h1>
        <!-- Vérifier si la liste n'est pas vide avant d'itérer -->
        <ul th:if="${not #lists.isEmpty(itemList)}">
            <li th:each="item : ${itemList}" th:text="${item}">Item</li>
        </ul>
        <p>Nombre d'éléments : <span th:text="${#lists.size(itemList)}">0</span></p>

        <!-- Vérifier la présence d'une clé dans une map -->
        <div th:if="${#maps.containsKey(featureMap, 'featureA')}">
            <p>Feature A est présente.</p>
        </div>
    </body>
    </html>
    ```
### Autres Utilitaires Notables

*   `#objects`: Utilitaires pour les objets génériques (ex: `isNull`, `isNotNull`, `defaultString`).
*   `#bools`: Utilitaires pour les opérations booléennes.
*   `#aggregates`: Fonctions d'agrégation sur des collections (ex: `sum`, `avg`).
*   `#ids`: Pour générer des ID uniques dans les éléments HTML (utile pour les itérations et le ciblage JS/CSS).

### Bonnes Pratiques

1.  **Privilégier les Utilitaires pour la Présentation :** Utiliser les objets utilitaires pour le formatage (dates, nombres, devises), les vérifications simples (chaînes vides, tailles de listes) et la gestion des URLs/i18n.
2.  **Éviter la Logique Complexe dans les Templates :** Si une opération nécessite plusieurs étapes, des calculs complexes ou des appels à des services, il est préférable de la réaliser dans le contrôleur ou une couche de service Java et de passer le résultat final au modèle. Les templates doivent rester focalisés sur l'affichage des données.
3.  **Utiliser `#temporals` pour Java 8+ :** Pour la manipulation des types `java.time.*`, l'objet `#temporals` est plus adapté et puissant que `#dates` ou `#calendars`.
4.  **Toujours utiliser `@{...}` pour les URLs :** Cela garantit la portabilité de l'application (indépendance du context path) et facilite la gestion des paramètres.
5.  **Centraliser les Textes avec `#messages` :** Utiliser `#messages` pour tout texte visible par l'utilisateur final facilite grandement la maintenance et l'internationalisation.
6.  **Exploiter la Null-Safety :** Combiner les utilitaires (comme `#objects.isNull` ou `#strings.isEmpty`) avec l'opérateur de navigation sécurisée (`?.`) et l'opérateur Elvis (`?:`) de Thymeleaf pour gérer élégamment les valeurs nulles.
    ```html
    <!-- Affiche le nom de l'utilisateur ou 'Invité' si user ou user.name est null -->
    <p th:text="${user?.name ?: 'Invité'}">Utilisateur</p>

    <!-- Alternative avec #objects -->
    <p th:text="${#objects.nullSafe(user?.profile?.email, 'Email non fourni')}">Email</p>
    ```
7.  **Consulter la Documentation :** La liste complète des utilitaires et de leurs méthodes est disponible dans la documentation officielle de Thymeleaf. Le développeur devrait s'y référer pour découvrir des fonctionnalités plus avancées.

---

## La factorisation des templates

L'objectif principal des layouts est d'éviter la répétition de code HTML commun à plusieurs pages
(comme l'en-tête, le pied de page, la barre de navigation, les inclusions CSS/JS, etc.)
et de centraliser la structure générale de vos pages web.

Il existe principalement deux approches avec Thymeleaf :

1. **Utilisation du Dialecte de Layout Thymeleaf (Thymeleaf Layout Dialect) :** C'est l'approche **recommandée et la
   plus puissante**. Elle est spécifiquement conçue pour la gestion des layouts et utilise un mécanisme
   d'héritage/décoration.
2. **Utilisation de `th:insert`, `th:replace`, `th:include` :** Une approche plus simple, basée sur l'inclusion de
   fragments. Moins flexible pour des layouts complexes, mais utile pour des composants réutilisables.

---

### 1. Utilisation du Dialecte de Layout Thymeleaf (Recommandé)

Ce dialecte, développé par `Ultraq`, n'est pas inclus par défaut dans Thymeleaf Core mais s'intègre parfaitement.
Spring Boot le configure automatiquement, il suffit d'ajouter la dépendance.

#### **Ajout de la dépendance**

* **Maven (`pom.xml`) :**
  ```xml
  <dependency>
      <groupId>nz.net.ultraq.thymeleaf</groupId>
      <artifactId>thymeleaf-layout-dialect</artifactId>
      <!-- La version est gérée par le parent Spring Boot -->
  </dependency>
  ```

#### **Création du Template de Layout (Gabarit)**

Il faut Créer un fichier HTML qui servira de structure de base pour les pages.
Par exemple, `templates/layouts/default.html` :

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
<head>
    <meta charset="UTF-8">
    <!-- Titre par défaut, peut être surchargé -->
    <title layout:title-pattern="$LAYOUT_TITLE - $CONTENT_TITLE">
        Mon Application
    </title>

    <!-- CSS Communs -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.5/dist/css/bootstrap.min.css"
          rel="stylesheet" integrity="sha384-SgOJa3DmI69IUzQ2PVdRZhwQ+dy64/BUtbMJw1MZ8t5HZApcHrRKUc4W0kG879m7"
          crossorigin="anonymous">
    <link rel="stylesheet" th:href="@{/css/style.css}"/>

    <!-- Bloc pour CSS spécifiques à la page (optionnel) -->
    <th:block layout:fragment="css"></th:block>
</head>
<body>
<!-- En-tête Commun -->
<header>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <!-- Contenu de la barre de navigation -->
        <a class="navbar-brand" th:href="@{/}">Mon App</a>
        <ul class="navbar-nav mr-auto">
            <li class="nav-item"><a class="nav-link" th:href="@{/}">Accueil</a></li>
            <li class="nav-item"><a class="nav-link" th:href="@{/about}">À Propos</a></li>
        </ul>
    </nav>
</header>

<!-- Section Principale - Contenu spécifique de la page -->
<main class="container mt-4">
    <!-- C'est ici que le contenu de la page spécifique sera injecté -->
    <div layout:fragment="content">
        <p>Contenu par défaut si la page enfant ne définit pas ce fragment.</p>
    </div>
</main>

<!-- Pied de page Commun -->
<footer class="text-center mt-5">
    <p>© 2025 Mon Application. Tous droits réservés.</p>
</footer>

<!-- Scripts JS Communs -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.5/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-k6d4wzSIapyDyv1kpU366/PK5hCdSbCRGRCMv+eplOQJWyd1fbcAu9OCUj5zNLiq"
        crossorigin="anonymous"></script>

<!-- Bloc pour Scripts spécifiques à la page (optionnel) -->
<th:block layout:fragment="scripts"></th:block>

</body>
</html>
```

#### **Explication des attributs clés :** {id="explication-des-attributs-cl-s_2"}

* `xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"`: Déclare le namespace pour utiliser les attributs du
  dialecte de layout (`layout:`).
* `layout:title-pattern="$LAYOUT_TITLE - $CONTENT_TITLE"`: (Optionnel) Permet de définir un modèle pour le titre de la
  page. `$LAYOUT_TITLE` est le titre défini dans ce layout, `$CONTENT_TITLE` est celui défini dans la page de contenu.
* `layout:fragment="content"`: **C'est le point crucial.** Il définit une zone remplaçable. Le contenu de la page
  spécifique (celle qui utilise ce layout) qui est marqué avec le même nom de fragment (`content` dans ce cas) viendra
  remplacer le contenu de cette `div`.
* `th:block layout:fragment="css"` et `th:block layout:fragment="scripts"`: Ce sont d'autres fragments *optionnels*. Les
  pages de contenu peuvent *choisir* de fournir du contenu pour ces fragments (par exemple, pour ajouter des CSS ou JS
  spécifiques à une page). `th:block` est une balise Thymeleaf qui disparaît après traitement, pratique pour ne pas
  ajouter de HTML superflu.

#### **Création des Pages de Contenu (Vues Spécifiques)**

Il faut Créer maintenant les pages qui utiliseront ce layout. Par exemple, `templates/home.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{layouts/default}">  <!-- 1. Indique quel layout utiliser -->
<head>
    <!-- 2. Définit le titre spécifique pour cette page -->
    <title>Page d'Accueil</title>

    <!-- 3. (Optionnel) Ajoute des CSS spécifiques -->
    <th:block layout:fragment="css">
        <link rel="stylesheet" th:href="@{/css/home-specific.css}"/>
    </th:block>
</head>
<body>

<!-- 4. Définit le contenu principal de la page -->
<div layout:fragment="content">
    <h1>Bienvenue sur la page d'accueil !</h1>
    <p>Ceci est le contenu spécifique à la page d'accueil.</p>
    <p>Le modèle Spring peut passer des données ici : <span th:text="${message}"></span></p>
</div>

<!-- 5. (Optionnel) Ajoute des scripts spécifiques -->
<th:block layout:fragment="scripts">
    <script th:src="@{/js/home-specific.js}"></script>
</th:block>

</body>
</html>
```

#### **Explication des attributs clés :** {id="explication-des-attributs-cl-s_1"}

* `layout:decorate="~{layouts/default}"`: **C'est l'instruction principale.** Elle indique que cette page "décore" (
  utilise) le layout situé à `templates/layouts/default.html`. Le `~{...}` est la syntaxe standard de Thymeleaf pour
  référencer d'autres templates.
* `<title>Page d'Accueil</title>`: Thymeleaf Layout Dialect est assez intelligent pour extraire ce titre et l'utiliser (
  par exemple avec `layout:title-pattern`).
* `layout:fragment="content"`: Le contenu de cette `div` remplacera la `div` ayant `layout:fragment="content"` dans le
  fichier `layouts/default.html`.
* `th:block layout:fragment="css"` et `th:block layout:fragment="scripts"`: Ces blocs définissent le contenu pour les
  fragments optionnels `css` et `scripts` définis dans le layout. S'ils n'étaient pas présents ici, les blocs
  correspondants dans le layout resteraient vides.

#### **Contrôleur Spring**

Le contrôleur Spring reste simple. Il suffit de retourner le nom de la vue de contenu (sans le chemin `templates/` ni
l'extension `.html`, car c'est géré par la configuration de Spring Boot et Thymeleaf).

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("message", "Donnée venant du contrôleur !");
        // Retourne le nom du template de contenu : templates/home.html
        // Thymeleaf et le Layout Dialect assembleront automatiquement
        // home.html avec layouts/default.html
        return "home";
    }

    @GetMapping("/about")
    public String about() {
        // Retourne le nom du template : templates/about.html
        // qui devra aussi utiliser layout:decorate="~{layouts/default}"
        return "about";
    }
}
```

#### **Fonctionnement**

Lorsque le contrôleur retourne `"home"`, Spring Boot et Thymeleaf localisent `templates/home.html`. Le Layout Dialect
voit l'attribut `layout:decorate="~{layouts/default}"`, charge alors `layouts/default.html`, puis remplace les
`layout:fragment` du layout par ceux trouvés dans `home.html`. Le HTML final généré et envoyé au navigateur est une
combinaison des deux fichiers.

---

### Utilisation de `th:insert`, `th:replace`, `th:include`

Cette approche n'utilise pas de dialecte externe. Elle repose sur la capacité de Thymeleaf à inclure des morceaux
(fragments) d'autres fichiers.

#### **Création de Fragments Réutilisables**

Il faut Créer des fichiers HTML contenant uniquement les parties réutilisables, en utilisant `th:fragment` pour les
nommer.

* `templates/fragments/header.html`:

  ```html
  <!DOCTYPE html>
  <html xmlns:th="http://www.thymeleaf.org">
  <body>
      <!-- Le fragment est nommé "mainHeader" -->
      <header th:fragment="mainHeader">
          <nav class="navbar navbar-expand-lg navbar-light bg-light">
              <a class="navbar-brand" th:href="@{/}">Mon App</a>
               <!-- ... reste de la nav ... -->
          </nav>
          <hr/>
      </header>
  </body>
  </html>
  ```

* `templates/fragments/footer.html`:
  ```html
  <!DOCTYPE html>
  <html xmlns:th="http://www.thymeleaf.org">
  <body>
      <!-- Le fragment est nommé "mainFooter" -->
      <footer th:fragment="mainFooter" class="text-center mt-5">
           <hr/>
          <p>&copy; 2023 Mon Application. Tous droits réservés.</p>
      </footer>
  </body>
  </html>
  ```

#### **Création de la Page Principale**

Chaque page doit explicitement inclure les fragments nécessaires.

* `templates/product_list.html`:
  ```html
  <!DOCTYPE html>
  <html xmlns:th="http://www.thymeleaf.org">
  <head>
      <meta charset="UTF-8">
      <title>Liste des Produits</title>
      <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.5/dist/css/bootstrap.min.css" 
            rel="stylesheet" 
            integrity="sha384-SgOJa3DmI69IUzQ2PVdRZhwQ+dy64/BUtbMJw1MZ8t5HZApcHrRKUc4W0kG879m7" 
            crossorigin="anonymous">

      <link rel="stylesheet" th:href="@{/css/style.css}" />
  </head>
  <body>
      <!-- Inclure et remplacer le tag actuel par le contenu du fragment header -->
      <div th:replace="~{fragments/header :: mainHeader}"></div>

      <main class="container mt-4">
          <h1>Nos Produits</h1>
          <!-- Contenu spécifique à la liste des produits -->
          <ul>
              <li th:each="product : ${products}" th:text="${product.name}">Produit</li>
          </ul>
      </main>

      <!-- Inclure et remplacer le tag actuel par le contenu du fragment footer -->
      <div th:replace="~{fragments/footer :: mainFooter}"></div>

      <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.5/dist/js/bootstrap.bundle.min.js" integrity="sha384-k6d4wzSIapyDyv1kpU366/PK5hCdSbCRGRCMv+eplOQJWyd1fbcAu9OCUj5zNLiq" crossorigin="anonymous"></script>
  </body>
  </html>
  ```

#### **Explication des attributs clés :**

* `th:fragment="fragmentName"`: Définit un morceau de code réutilisable dans un fichier de fragments.
* `th:replace="~{chemin/vers/fichier :: nomDuFragment}"`: Remplace la balise hôte (ici la `<div>`) par le fragment
  spécifié. C'est généralement préféré pour l'inclusion de composants.
* `th:insert="~{chemin/vers/fichier :: nomDuFragment}"`: Insère le fragment *à l'intérieur* de la balise hôte. La balise
  hôte est conservée.
* `th:include="~{chemin/vers/fichier :: nomDuFragment}"`: (Plus ancien, similaire à `th:insert` mais insère uniquement
  le *contenu* du fragment, pas la balise du fragment elle-même). `th:insert` ou `th:replace` sont souvent plus clairs.

#### **Inconvénients pour les Layouts Complets**

* **Répétition :** La structure de base (`<html>`, `<head>`, `<body>`, inclusions CSS/JS communes) doit être répétée ou
  incluse dans *chaque* page principale.
* **Moins élégant :** La logique d'assemblage est répartie dans chaque page de contenu plutôt que centralisée dans un
  layout parent.
* **Plus verbeux :** Chaque page doit explicitement lister tous les fragments communs qu'elle utilise.

**Quand utiliser cette approche ?**
Elle est excellente pour des **composants réutilisables plus petits** au sein d'une page (une carte produit, un
formulaire de recherche, une modale) plutôt que pour la structure globale de la page.

---

### Conclusion

* Pour une gestion **robuste et maintenable des layouts de page**, il faut utiliser le **Thymeleaf Layout Dialect** (
  `layout:decorate`, `layout:fragment`). C'est la méthode standard et recommandée dans l'écosystème Spring/Thymeleaf.
* Pour inclure des **petits composants réutilisables** (fragments) à différents endroits de vos pages, on peut utiliser
  `th:replace` (ou `th:insert`) avec `th:fragment`.

Il est bien entendu possible de combiner les deux approches.

### Héritage et surcharge des blocs

Par défaut un `layout:fragment` dans une page enfant **remplace** entièrement le contenu du `layout:fragment`
correspondant dans le layout parent.

Cependant, Il est possible de **combiner** le contenu en utilisant une balise spéciale `layout:fragment` (ou `th:block 
layout:fragment`) *à l'intérieur* de la définition du fragment de l'enfant.
Cette balise spéciale agit comme un **placeholder** pour le contenu original du fragment parent.

Voici comment cela fonctionne :

**Scénario :**

* **Layout Parent (`templates/layouts/default-override.html`) :**
  ```html
  <!DOCTYPE html>
  <html xmlns:th="http://www.thymeleaf.org"
        xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
  <head>
      <title layout:title-pattern="$CONTENT_TITLE | $LAYOUT_TITLE">Mon Layout</title>

      <!-- Bloc pour scripts additionnels -->
      <th:block layout:fragment="pageScripts">
          <!-- 
              Contenu par défaut 
              (peut être vide ou contenir des scripts 
              communs chargés en dernier) 
          -->
          <script th:src="@{/js/common.js}"></script>
      </th:block>
  </head>
  <body>
      <header>...</header>
      <main layout:fragment="content">
          <p>Contenu par défaut du layout.</p>
      </main>
      <footer>...</footer>
  </body>
  </html>
  ```

* **Page Enfant (`templates/myPage.html`) :**
```html
  <!DOCTYPE html>
  <html xmlns:th="http://www.thymeleaf.org"
        xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
        layout:decorate="~{layouts/default-override}">
  <head>
      <title>Ma Page Spécifique</title>
  </head>
  <body>

      <!-- Exemple 1: Ajouter APRÈS le contenu parent -->
      <div layout:fragment="content">
          <!-- Contenu spécifique à cette page -->
          <h1>Titre de Ma Page</h1>
          <p>Ceci est mon contenu spécifique.</p>

          <!-- Insérer ici le contenu original du fragment 'content' du parent -->
          <th:block layout:fragment="content"></th:block>
          <!-- <layout:fragment/> est aussi une syntaxe valide -->

           <p>Du contenu supplémentaire après celui du parent.</p>
      </div>


      <!-- Exemple 2: Ajouter AVANT le contenu parent (pour les scripts) -->
      <th:block layout:fragment="pageScripts">
          <!-- Script spécifique à cette page -->
          <script th:src="@{/js/myPage-specific.js}"></script>

          <!-- Insérer ici le contenu original du fragment 'pageScripts' du parent -->
          <th:block layout:fragment="pageScripts"></th:block>
          <!-- <layout:fragment/> fonctionne aussi -->
      </th:block>

  </body>
  </html>
```

**Explication :**

1. **`layout:fragment="content"` dans `myPage.html` :**

* Il définit le contenu pour le fragment `content`.
* À l'intérieur, la ligne `<th:block layout:fragment="content"></th:block>` (ou `<layout:fragment/>`) dit au Layout
  Dialect : "À cet endroit précis, insère le contenu original qui se trouvait dans le `layout:fragment="content"` du
  fichier `default.html`".
* Résultat : Le rendu final combinera le contenu spécifique de `myPage.html` avec le
  `<p>Contenu par défaut du layout.</p>` du parent, en respectant l'ordre défini dans l'enfant. Dans cet exemple, le
  contenu du parent sera inséré *entre* le contenu spécifique de l'enfant.

2. **`layout:fragment="pageScripts"` dans `myPage.html` :**

* Il définit le contenu pour le fragment `pageScripts`.
* Il ajoute d'abord le script `myPage-specific.js`.
* Ensuite, `<th:block layout:fragment="pageScripts"></th:block>` insère le contenu original du fragment `pageScripts` du
  parent (qui est `<script th:src="@{/js/common-late-load.js}"></script>`).
* Résultat : Le script spécifique à la page sera chargé *avant* le script `common-late-load.js` défini dans le layout.

**Cas d'utilisation courants :**

* **Ajouter des classes CSS au `<body>` :** Le layout parent peut avoir `<body layout:fragment="bodyAttributes">`, et
  l'enfant peut faire `<body layout:fragment="bodyAttributes" class="page-specific-class"> <layout:fragment/> </body>`
  pour ajouter une classe sans perdre d'autres attributs éventuels du parent.
* **Scripts JS :** Charger des scripts communs dans le layout et ajouter des scripts spécifiques à la page avant ou
  après les communs.
* **Meta tags :** Ajouter des meta tags spécifiques à une page tout en conservant ceux définis dans le layout.
* **Contenu par défaut :** Fournir un contenu par défaut dans le layout (par exemple, un message "Aucun contenu fourni")
  qui peut être soit remplacé, soit complété par l'enfant.

En résumé, l'utilisation de `<layout:fragment/>` ou `<th:block layout:fragment/>` (avec le même nom de fragment) à
l'intérieur de la définition du fragment enfant donne un contrôle total sur la manière dont le contenu enfant et le
contenu parent sont fusionnés, allant au-delà du simple remplacement.

## Exercices

