# Les sessions

Dans le développement d'applications web, la gestion de l'état entre plusieurs requêtes d'un même utilisateur est une
nécessité courante. Le protocole HTTP étant intrinsèquement sans état (stateless), chaque requête est indépendante des
précédentes. Les sessions offrent un mécanisme pour pallier cette limitation en permettant de stocker des informations
spécifiques à un utilisateur sur le serveur pendant la durée de sa visite.

Spring Boot, en s'appuyant sur l'API Servlet standard, simplifie grandement la gestion des sessions HTTP. Ce support de
cours explore comment utiliser efficacement et sécuritairement les sessions dans une application web Spring Boot (par
exemple, avec Thymeleaf ou JSP), en excluant le contexte des API REST qui utilisent généralement d'autres mécanismes d'
état (comme les tokens JWT).

## Concepts Fondamentaux de la Session HTTP

### Qu'est-ce qu'une Session ?

Une session représente une conversation entre un client (navigateur) et le serveur. Elle commence lorsqu'un utilisateur
accède à l'application pour la première fois et se termine après une période d'inactivité ou lorsque l'utilisateur se
déconnecte explicitement.

### Fonctionnement Technique (Simplifié)

1. **Première Requête :** L'utilisateur accède à l'application. Le serveur ne trouve pas d'identifiant de session
   existant.
2. **Création de Session :** Le serveur crée un nouvel objet `HttpSession` en mémoire.
3. **Génération de l'ID de Session :** Un identifiant unique (Session ID) est généré pour cette session.
4. **Envoi du Cookie :** Le serveur envoie cet ID de session au client via un cookie (généralement nommé `JSESSIONID`
   par défaut dans les serveurs d'applications Java).
5. **Requêtes Suivantes :** Le navigateur du client inclut automatiquement ce cookie `JSESSIONID` dans chaque requête
   ultérieure vers le même domaine.
6. **Récupération de la Session :** Le serveur utilise l'ID reçu via le cookie pour retrouver l'objet `HttpSession`
   correspondant en mémoire et ainsi accéder aux données stockées pour cet utilisateur.

### L'objet `HttpSession`

L'interface `jakarta.servlet.http.HttpSession` (ou `javax.servlet.http.HttpSession` pour les anciennes versions de
Servlet API) est au cœur de la gestion de session en Java EE / Jakarta EE. Elle permet principalement de :

* Stocker des données sous forme de paires clé/valeur (`setAttribute(String name, Object value)`).
* Récupérer des données (`getAttribute(String name)`).
* Supprimer des données (`removeAttribute(String name)`).
* Obtenir l'identifiant unique de la session (`getId()`).
* Invalider (terminer) la session (`invalidate()`).
* Gérer le délai d'inactivité (`setMaxInactiveInterval(int interval)`).

Spring Boot gère la création et la mise à disposition de cet objet `HttpSession` de manière transparente.

## Utilisation Basique des Sessions dans un Contrôleur Spring Boot

Il existe plusieurs façons d'accéder et de manipuler la session HTTP dans un contrôleur Spring Boot.

### Accès via `HttpServletRequest`

On peut obtenir la session à partir de l'objet `HttpServletRequest` injecté dans la méthode du contrôleur.

```java
package fr.formation.spring.controller;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class BasicSessionController {

    @GetMapping("/compteur-visites")
    public String countVisits(HttpServletRequest request, Model model) {
        // Obtient la session, la crée si elle n'existe pas (true)
        HttpSession session = request.getSession(true);

        // Récupère le compteur actuel de la session
        Integer visitCount = (Integer) session.getAttribute("visitCounter");

        if (visitCount == null) {
            // Si le compteur n'existe pas, l'initialiser à 1
            visitCount = 1;
        } else {
            // Sinon, l'incrémenter
            visitCount++;
        }

        // Met à jour le compteur dans la session
        session.setAttribute("visitCounter", visitCount);

        // Ajoute le compteur au modèle pour l'afficher dans la vue
        model.addAttribute("count", visitCount);

        // Retourne le nom de la vue (ex: compteur-visites.html)
        return "compteur-visites";
    }

    @GetMapping("/reset-compteur")
    public String resetCounter(HttpServletRequest request) {
        HttpSession session = request.getSession(false); // Ne crée pas si n'existe pas
        if (session != null) {
            // Supprime l'attribut spécifique
            session.removeAttribute("visitCounter");
        }
        // Redirige vers la page du compteur
        return "redirect:/compteur-visites";
    }
}
```

* `request.getSession(true)` : Obtient la session courante. Si aucune session n'existe pour cette requête, une nouvelle
  session est créée.
* `request.getSession(false)` : Obtient la session courante. Si aucune session n'existe, retourne `null` (utile pour
  vérifier si une session est active sans en créer une).
* `session.setAttribute(name, value)` : Stocke un objet dans la session.
* `session.getAttribute(name)` : Récupère un objet de la session (nécessite un cast).
* `session.removeAttribute(name)` : Supprime un attribut de la session.

### Injection directe de `HttpSession`

Spring MVC permet d'injecter directement l'objet `HttpSession` comme paramètre d'une méthode de contrôleur. C'est
souvent considéré comme plus propre.

```java
package fr.formation.spring.controller;

import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class InjectedSessionController {

    @GetMapping("/info-session")
    public String showSessionInfo(HttpSession session, Model model) {
        // Récupère l'ID de la session
        String sessionId = session.getId();

        // Récupère un attribut (exemple: nom d'utilisateur stocké ailleurs)
        String username = (String) session.getAttribute("loggedInUser");

        if (username == null) {
            username = "Visiteur anonyme";
        }

        model.addAttribute("sessionId", sessionId);
        model.addAttribute("user", username);

        return "info-session"; // Nom de la vue (info-session.html)
    }

    @GetMapping("/terminer-session")
    public String endSession(HttpSession session) {
        // Invalide la session complète
        session.invalidate();
        // Redirige vers une page d'accueil ou de confirmation
        return "redirect:/";
    }
}
```

* L'injection de `HttpSession` est gérée par Spring. Si aucune session n'existe, Spring en créera une automatiquement (
  équivalent à `request.getSession(true)`).
* `session.invalidate()` : Termine la session. Tous les attributs sont supprimés, et l'ID de session n'est plus valide.
  Une nouvelle session sera créée lors de la prochaine requête nécessitant une session.

## Beans de Portée Session (`@SessionScope`)

Une approche plus orientée objet et souvent préférable pour gérer l'état lié à la session consiste à utiliser des beans
Spring avec la portée `session`. Un bean `@SessionScope` est une instance unique par session HTTP. Spring gère le cycle
de vie de ce bean et le lie automatiquement à la session de l'utilisateur courant.

### Déclaration d'un Bean `@SessionScope`

```java
package fr.formation.spring.model;

import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.SessionScope;

import java.io.Serializable; // Important pour la sérialisation potentielle
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Component
@SessionScope // Définit la portée de ce bean à la session HTTP
public class ShoppingCart implements Serializable {

    private static final long serialVersionUID = 1L; // Bonne pratique pour Serializable

    // Liste des articles dans le panier
    private final List<String> items = new ArrayList<>();

    public void addItem(String item) {
        this.items.add(item);
    }

    public List<String> getItems() {
        // Retourne une copie non modifiable pour l'extérieur
        return Collections.unmodifiableList(this.items);
    }

    public void clearCart() {
        this.items.clear();
    }

    public int getItemCount() {
        return this.items.size();
    }
}
```

* `@Component` (ou `@Service`, `@Repository`) : Marque la classe comme un bean Spring.
* `@SessionScope` : Indique que Spring doit créer une instance de ce bean par session HTTP active. L'instance sera
  stockée comme un attribut de session.
* `implements Serializable` : Bien que non strictement requis pour le stockage en mémoire par défaut de Tomcat, c'est
  une **bonne pratique** et devient **nécessaire** si vous configurez la persistance de session (par exemple, en base de
  données ou Redis) ou dans un environnement clusterisé où les sessions peuvent être sérialisées et transférées.

### Injection et Utilisation dans un Contrôleur

Ce bean peut ensuite être injecté directement dans les contrôleurs (ou d'autres beans). Spring fournira l'instance
correcte associée à la session de l'utilisateur effectuant la requête.

```java
package fr.formation.spring.controller;

import fr.formation.spring.model.ShoppingCart;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class CartController {

    // Injection du bean de portée session
    private final ShoppingCart shoppingCart;

    @Autowired // L'injection par constructeur est recommandée
    public CartController(ShoppingCart shoppingCart) {
        this.shoppingCart = shoppingCart;
    }

    @GetMapping("/panier")
    public String viewCart(Model model) {
        // Utilise directement le bean injecté
        model.addAttribute("items", shoppingCart.getItems());
        model.addAttribute("itemCount", shoppingCart.getItemCount());
        return "panier"; // Vue panier.html
    }

    @PostMapping("/panier/ajouter")
    public String addToCart(@RequestParam String item) {
        // Ajoute un article via le bean de session
        if (item != null && !item.trim().isEmpty()) {
            shoppingCart.addItem(item.trim());
        }
        // Redirige vers la vue du panier après l'ajout
        return "redirect:/panier";
    }

    @PostMapping("/panier/vider")
    public String clearCart() {
        // Vide le panier via le bean de session
        shoppingCart.clearCart();
        return "redirect:/panier";
    }
}
```

**Comment ça marche en interne ?** Spring n'injecte pas directement l'instance du bean `ShoppingCart` dans le
contrôleur (qui est généralement un singleton). Il injecte un **proxy** qui cible l'instance `ShoppingCart` réelle
associée à la session de la requête courante. Cela permet d'avoir un contrôleur singleton qui peut interagir avec des
beans de portée session spécifiques à chaque utilisateur.

## Cas d'Utilisation Courants

1. **Gestion de l'authentification utilisateur :** Stocker des informations sur l'utilisateur connecté (ID, nom
   d'utilisateur, rôles) après une connexion réussie. *Attention : Ne jamais stocker le mot de passe, même hashé, en
   session.*
2. **Panier d'achat :** Maintenir la liste des articles sélectionnés par un utilisateur avant la validation de la
   commande. Le bean `@SessionScope` est idéal ici.
3. **Préférences utilisateur :** Conserver des choix temporaires comme la langue, le thème d'affichage, ou des filtres
   de recherche pendant la navigation.
4. **Données de formulaire multi-étapes :** Stocker les informations saisies dans les étapes précédentes d'un formulaire
   complexe ("wizard").
5. **Messages Flash :** Afficher des messages (succès, erreur) après une redirection (voir section suivante).


## Sécurité des Sessions : Points Cruciaux

La gestion des sessions est une cible fréquente d'attaques. Il est **impératif** de sécuriser leur utilisation.

### Fixation de Session (Session Fixation)

* **Attaque :** L'attaquant force le navigateur de la victime à utiliser un ID de session qu'il connaît *avant* que la
  victime ne s'authentifie. Une fois la victime authentifiée, l'attaquant peut utiliser cet ID de session connu pour
  usurper la session authentifiée.
* **Contre-mesure :** **Toujours** régénérer l'ID de session **immédiatement après une authentification réussie** (et
  idéalement lors de tout changement de niveau de privilège). Spring Security le fait automatiquement. Si vous gérez
  l'authentification manuellement :

  ```java
  import jakarta.servlet.http.HttpServletRequest;
  // ... dans la méthode de login après vérification des identifiants ...
  public String loginSuccess(HttpServletRequest request) {
      // ... vérifier mot de passe ...
      
      // Régénère l'ID de session, invalide l'ancien
      request.changeSessionId(); // Très important !
      
      // Stocker les infos utilisateur dans la *nouvelle* session
      HttpSession newSession = request.getSession();
      newSession.setAttribute("loggedInUser", "username"); 
      
      return "redirect:/accueil-connecte";
  }
  ```

### Vol de Session (Session Hijacking)

* **Attaque :** L'attaquant obtient l'ID de session valide d'une victime (par sniffing réseau, XSS, etc.) et l'utilise
  pour accéder à l'application en se faisant passer pour la victime.
* **Contre-mesures :**
    * **Utiliser HTTPS :** Chiffre la communication entre le client et le serveur, y compris le cookie de session,
      empêchant le sniffing sur des réseaux non sécurisés. C'est la mesure la plus importante.
    * **Cookie `HttpOnly` :** Empêche l'accès au cookie de session via JavaScript côté client. Cela mitige grandement le
      risque de vol via des attaques XSS (Cross-Site Scripting). Spring Boot active `HttpOnly` par défaut pour le cookie
      de session. Vérifiez et forcez-le si nécessaire :
      ```properties
      # Dans application.properties
      server.servlet.session.cookie.http-only=true 
      ```
    * **Cookie `Secure` :** Indique au navigateur de n'envoyer le cookie que sur des connexions HTTPS. Spring Boot
      l'active automatiquement si HTTPS est détecté ou peut être forcé :
      ```properties
      # Dans application.properties
      server.servlet.session.cookie.secure=true 
      ```
      Nécessite que votre application soit servie via HTTPS.

### Expiration de Session (Session Timeout)

* **Risque :** Une session qui reste active trop longtemps après une période d'inactivité augmente la fenêtre d'opportunité pour un attaquant si l'ID de session est compromis ou si l'ordinateur de la victime est laissé sans
  surveillance.
* **Contre-mesure :** Configurer un délai d'expiration raisonnable (par exemple, 15-30 minutes d'inactivité).
  ```properties
  # Dans application.properties
  # Délai en secondes (ici, 30 minutes)
  server.servlet.session.timeout=1800 
  # Ou en utilisant la notation Duration (plus lisible)
  # server.servlet.session.timeout=30m 
  ```
* **Déconnexion explicite :** Toujours fournir un bouton/lien de déconnexion qui appelle `session.invalidate()`.

### Stockage de Données Sensibles

* **Risque :** Stocker des informations hautement sensibles (mots de passe, numéros de carte de crédit complets, clés
  API privées) directement en session est **fortement déconseillé**. Même si la session elle-même est sécurisée, cela
  augmente la surface d'attaque potentielle (accès mémoire du serveur, logs accidentels, erreurs de développement).
* **Bonne Pratique :** Ne stockez en session que des identifiants (ID utilisateur, ID panier), des états non critiques (
  préférences), ou des données temporaires non sensibles. Récupérez les données sensibles depuis la base de données
  lorsque c'est nécessaire, en utilisant l'ID utilisateur stocké en session.

### CSRF (Cross-Site Request Forgery)

Bien que non directement lié au *vol* de session, CSRF exploite une session existante. L'attaquant incite la victime
authentifiée à exécuter involontairement une action sur l'application (ex: via un lien ou formulaire sur un autre site).

* **Contre-mesure :** Utiliser des tokens anti-CSRF. Spring Security fournit une protection robuste contre CSRF, qui est
  activée par défaut. Si vous n'utilisez pas Spring Security, vous devrez implémenter cette protection manuellement.

## Configuration Avancée (via `application.properties`)

Spring Boot permet de configurer divers aspects de la gestion de session :

```properties
# Configuration de la Session Servlet
# Durée d'inactivité maximale avant expiration (format Duration)
server.servlet.session.timeout=30m 
# Nom du cookie de session (défaut: JSESSIONID)
# Changer peut légèrement obscurcir, mais pas une mesure de sécurité forte
# server.servlet.session.cookie.name=MYSESSIONID 
# Le cookie est-il accessible uniquement via HTTP (non JS) ? (Défaut: true)
server.servlet.session.cookie.http-only=true
# Le cookie doit-il être envoyé uniquement via HTTPS ? (Défaut: false, mais activé si SSL/TLS est configuré)
# À mettre à true si l'application est toujours servie en HTTPS
server.servlet.session.cookie.secure=true 
# Chemin pour lequel le cookie est valide (Défaut: /)
# server.servlet.session.cookie.path=/ 
# Domaine pour lequel le cookie est valide (utile pour les sous-domaines)
# server.servlet.session.cookie.domain=example.com 
# Âge maximum du cookie (en secondes). -1 signifie cookie de session (expire à la fermeture du navigateur)
# Une valeur positive crée un cookie persistant.
# server.servlet.session.cookie.max-age=-1 
# Stratégie de suivi de session (cookie, url, ssl) - 'cookie' est le standard
# server.servlet.session.tracking-modes=cookie 
```

## Bonnes Pratiques et Conseils

1. **Utiliser avec Parcimonie :** Les sessions consomment de la mémoire serveur. Ne stockez que ce qui est strictement
   nécessaire.
2. **Préférer les Beans `@SessionScope` :** Pour l'état complexe lié à la session (panier, profil utilisateur), ils
   offrent une meilleure encapsulation et testabilité que la manipulation directe d'attributs `HttpSession`.
3. **Attention à la Taille des Objets :** Évitez de stocker des objets volumineux en session. Cela impacte la mémoire et
   peut poser problème avec la réplication ou la persistance de session. Stockez des identifiants et rechargez les
   données si besoin.
4. **Implémenter `Serializable` :** Même si ce n'est pas toujours requis, c'est une bonne pratique pour les objets
   stockés en session, surtout pour les beans `@SessionScope`.
5. **Sécuriser :** Appliquez **toutes** les mesures de sécurité mentionnées (HTTPS, HttpOnly, Secure, régénération d'ID,
   timeouts courts, pas de données sensibles). Utilisez Spring Security pour une gestion robuste de l'authentification
   et de la sécurité des sessions.
6. **Nettoyer :** Utilisez `session.removeAttribute()` lorsque les données ne sont plus nécessaires, et
   `session.invalidate()` lors de la déconnexion pour libérer les ressources.
7. **Utiliser les Attributs Flash :** Pour les messages post-redirection, préférez
   `RedirectAttributes.addFlashAttribute()` à la manipulation manuelle de la session.
8. **Tester :** Testez le comportement de votre application avec des sessions expirées, des déconnexions, et essayez de
   simuler des scénarios de sécurité.

##. Exercices Pratiques

### Exercice 1 : Compteur de Rafraîchissements

**Objectif :** Utiliser `HttpSession` directement pour compter combien de fois un utilisateur a rafraîchi une page
spécifique.

**Énoncé :**

1. Créez un contrôleur `RefreshCounterController`.
2. Créez une méthode GET mappée sur `/compteur-refresh`.
3. Dans cette méthode, accédez à `HttpSession` (en l'injectant ou via `HttpServletRequest`).
4. Récupérez un attribut nommé `"refreshCount"`.
5. S'il n'existe pas, initialisez-le à 1. Sinon, incrémentez sa valeur.
6. Stockez la nouvelle valeur dans la session.
7. Ajoutez la valeur du compteur au `Model` pour l'affichage.
8. Créez une vue Thymeleaf (`compteur-refresh.html`) qui affiche : "Cette page a été rafraîchie X fois durant cette
   session."

#### **Correction Exercice 1** {collapsible="true"}

*Controller (`RefreshCounterController.java`):*

```java
package fr.formation.spring.controller;

import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class RefreshCounterController {

    @GetMapping("/compteur-refresh")
    public String countRefreshes(HttpSession session, Model model) {
        // Nom de l'attribut de session
        final String REFRESH_COUNT_ATTR = "refreshCount";

        // Récupère la valeur actuelle, peut être null
        Integer count = (Integer) session.getAttribute(REFRESH_COUNT_ATTR);

        if (count == null) {
            // Première visite/refresh sur cette page dans la session
            count = 1;
        } else {
            // Incrémente le compteur
            count++;
        }

        // Met à jour l'attribut dans la session
        session.setAttribute(REFRESH_COUNT_ATTR, count);

        // Ajoute au modèle pour la vue
        model.addAttribute("refreshNumber", count);

        return "compteur-refresh"; // Nom de la vue Thymeleaf
    }
}
```

*View (`templates/compteur-refresh.html`):*

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Compteur de Rafraîchissements</title>
</head>
<body>
<h1>Compteur de Rafraîchissements</h1>
<p>
    Cette page a été rafraîchie
    <strong th:text="${refreshNumber}">0</strong>
    fois durant cette session.
</p>
<p>
    <a th:href="@{/compteur-refresh}">Rafraîchir la page</a>
</p>
</body>
</html>
```

---

### Exercice 2 : Préférences Utilisateur avec Bean `@SessionScope`

**Objectif :** Utiliser un bean `@SessionScope` pour stocker et modifier une préférence utilisateur (par exemple, une
couleur de fond).

**Énoncé :**

1. Créez une classe `UserPreferences` annotée avec `@Component` et `@SessionScope`. Elle doit implémenter
   `Serializable`.
2. Ajoutez un champ privé `String backgroundColor` (avec "white" comme valeur par défaut) et les getters/setters
   correspondants.
3. Créez un contrôleur `PreferencesController`.
4. Injectez le bean `UserPreferences` dans ce contrôleur.
5. Créez une méthode GET mappée sur `/preferences` qui affiche la préférence actuelle et un formulaire pour la changer.
   La couleur de fond actuelle doit être ajoutée au `Model`.
6. Créez une méthode POST mappée sur `/preferences/save` qui reçoit la nouvelle couleur (`@RequestParam String color`).
   Mettez à jour la couleur dans le bean `UserPreferences`. Redirigez ensuite vers `/preferences`.
7. Créez une vue Thymeleaf (`preferences.html`) qui :
    * Affiche la couleur actuelle.
    * Utilise la couleur stockée pour définir le style de fond du `<body>` (par exemple,
      `style="background-color: couleur;"`).
    * Contient un formulaire simple (un champ texte + bouton submit) pour soumettre une nouvelle couleur à
      `/preferences/save`.

#### **Correction Exercice 2** {collapsible="true"}

*Bean (`UserPreferences.java`):*

```java
package fr.formation.spring.model;

import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.SessionScope;

import java.io.Serializable;

@Component
@SessionScope
public class UserPreferences implements Serializable {

    private static final long serialVersionUID = 1L;

    // Préférence de couleur de fond, blanc par défaut
    private String backgroundColor = "white";

    public String getBackgroundColor() {
        return backgroundColor;
    }

    public void setBackgroundColor(String backgroundColor) {
        // Simple validation: ne pas accepter de valeur vide ou nulle
        if (backgroundColor != null && !backgroundColor.trim().isEmpty()) {
            this.backgroundColor = backgroundColor.trim();
        }
    }
}
```

*Controller (`PreferencesController.java`):*

```java
package fr.formation.spring.controller;

import fr.formation.spring.model.UserPreferences;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class PreferencesController {

    private final UserPreferences userPreferences;

    @Autowired
    public PreferencesController(UserPreferences userPreferences) {
        this.userPreferences = userPreferences;
    }

    @GetMapping("/preferences")
    public String showPreferences(Model model) {
        // Ajoute la couleur actuelle au modèle pour la vue
        model.addAttribute("currentBackgroundColor",
                userPreferences.getBackgroundColor());
        // Ajoute l'objet entier si besoin dans la vue (ici pas nécessaire)
        // model.addAttribute("userPrefs", userPreferences); 
        return "preferences"; // Vue preferences.html
    }

    @PostMapping("/preferences/save")
    public String savePreferences(@RequestParam String color) {
        // Met à jour la préférence via le bean de session
        userPreferences.setBackgroundColor(color);
        // Redirige vers la page des préférences pour voir le changement
        return "redirect:/preferences";
    }
}
```

*View (`templates/preferences.html`):*

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Préférences Utilisateur</title>
</head>
<!-- Applique le style de fond dynamiquement -->
<body th:style="'background-color:' + ${currentBackgroundColor}">

<h1>Préférences Utilisateur</h1>

<p>
    La couleur de fond actuelle est :
    <strong th:text="${currentBackgroundColor}">white</strong>.
</p>

<h2>Changer la couleur de fond</h2>
<form th:action="@{/preferences/save}" method="post">
    <label for="colorInput">Nouvelle couleur (nom ou code hex):</label>
    <input type="text" id="colorInput" name="color"
           th:value="${currentBackgroundColor}" required>
    <button type="submit">Sauvegarder</button>
</form>

<p>
    <a th:href="@{/}">Retour à l'accueil</a>
</p>

</body>
</html>
```

---

### Exercice 3 : Simulation de Connexion/Déconnexion et Sécurité

**Objectif :** Simuler une connexion utilisateur, stocker le nom d'utilisateur en session, l'afficher, et implémenter la
déconnexion en invalidant la session. Insister sur la régénération de l'ID de session.

**Énoncé :**

1. Créez un contrôleur `AuthController`.
2. Créez une méthode GET pour `/login` qui affiche un formulaire de connexion simple (champ `username` uniquement, pas
   de mot de passe pour simplifier).
3. Créez une méthode POST pour `/login` qui reçoit le `username`.
    * Simulez une authentification réussie si le `username` n'est pas vide.
    * **Important :** Obtenez `HttpServletRequest` et appelez `request.changeSessionId()` pour régénérer l'ID de
      session.
    * Obtenez la *nouvelle* session (via `request.getSession()`) et stockez le `username` dans un attribut
      `"loggedInUser"`.
    * Redirigez vers une page `/accueil`.
4. Créez une méthode GET pour `/accueil`.
    * Accédez à la session.
    * Récupérez l'attribut `"loggedInUser"`.
    * Si l'utilisateur est connecté (attribut non null), affichez "Bienvenue, [username] !" avec un lien/bouton de
      déconnexion.
    * Sinon, affichez "Veuillez vous connecter." avec un lien vers `/login`.
5. Créez une méthode GET ou POST pour `/logout`.
    * Accédez à la session.
    * Appelez `session.invalidate()`.
    * Redirigez vers `/accueil`.
6. Créez les vues Thymeleaf nécessaires (`login.html`, `accueil.html`).

#### **Correction Exercice 3** {collapsible="true"}

*Controller (`AuthController.java`):*

```java
package fr.formation.spring.controller;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class AuthController {

    // Affiche le formulaire de login
    @GetMapping("/login")
    public String showLoginForm() {
        return "login"; // Vue login.html
    }

    // Traite la soumission du formulaire de login
    @PostMapping("/login")
    public String handleLogin(@RequestParam String username,
                              HttpServletRequest request) {

        // Vérification très basique (ne pas faire ça en production !)
        if (username != null && !username.trim().isEmpty()) {
            // *** Sécurité : Régénérer l'ID de session après login ***
            request.changeSessionId();

            // Obtenir la *nouvelle* session après régénération
            HttpSession session = request.getSession();

            // Stocker l'info utilisateur dans la session
            session.setAttribute("loggedInUser", username.trim());

            // Rediriger vers la page d'accueil sécurisée
            return "redirect:/accueil";
        } else {
            // Échec de la "connexion", rediriger vers le formulaire avec erreur
            // (Pour simplifier, on redirige juste vers login)
            return "redirect:/login?error";
        }
    }

    // Affiche la page d'accueil, change selon l'état de connexion
    @GetMapping("/accueil")
    public String showHome(HttpSession session, Model model) {
        String username = (String) session.getAttribute("loggedInUser");

        if (username != null) {
            // Utilisateur connecté
            model.addAttribute("username", username);
            return "accueil-connecte"; // Vue accueil-connecte.html
        } else {
            // Utilisateur non connecté
            return "accueil-public"; // Vue accueil-public.html
        }
    }

    // Gère la déconnexion
    @GetMapping("/logout")
    public String handleLogout(HttpSession session) {
        // Invalide la session
        session.invalidate();
        // Redirige vers la page publique
        return "redirect:/accueil";
    }
}
```

*View (`templates/login.html`):*

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Connexion</title></head>
<body>
<h1>Connexion</h1>
<!-- Affiche un message d'erreur si ?error est dans l'URL -->
<p th:if="${param.error}" style="color:red;">
    Nom d'utilisateur requis.
</p>
<form th:action="@{/login}" method="post">
    <div>
        <label for="username">Nom d'utilisateur :</label>
        <input type="text" id="username" name="username" required>
    </div>
    <button type="submit">Se connecter</button>
</form>
</body>
</html>
```

*View (`templates/accueil-connecte.html`):*

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Accueil</title></head>
<body>
<h1>Accueil</h1>
<p>
    Bienvenue, <strong th:text="${username}">Utilisateur</strong> !
</p>
<form th:action="@{/logout}" method="get"> <!-- Ou POST -->
    <button type="submit">Se déconnecter</button>
</form>
</body>
</html>
```

*View (`templates/accueil-public.html`):*

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Accueil</title></head>
<body>
<h1>Accueil</h1>
<p>Veuillez vous connecter pour accéder au contenu.</p>
<p>
    <a th:href="@{/login}">Se connecter</a>
</p>
</body>
</html>
```

*(Note : Pour simplifier, deux vues d'accueil ont été créées. On pourrait utiliser `th:if` dans une seule
vue `accueil.html`)*

---

## Conclusion

La gestion des sessions est un aspect fondamental des applications web stateful. Spring Boot facilite grandement
l'utilisation de `HttpSession` et propose une alternative élégante avec les beans `@SessionScope`. Cependant, la
simplicité d'utilisation ne doit pas occulter les **impératifs de sécurité**. La protection contre la fixation et le vol
de session, la gestion correcte des timeouts, et l'utilisation prudente des données stockées sont essentielles pour
créer des applications robustes et sécurisées. Pour des besoins d'authentification et d'autorisation complexes,
l'utilisation de **Spring Security** est fortement recommandée, car il intègre nativement la plupart de ces bonnes
pratiques de sécurité liées aux sessions.