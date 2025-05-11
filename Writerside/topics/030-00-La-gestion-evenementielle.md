# La gestion événementielle

Dans une application, en particulier une application web, de nombreuses actions peuvent nécessiter des traitements
secondaires ou des notifications. Par exemple, après l'inscription d'un utilisateur, il peut être nécessaire d'envoyer
un email de bienvenue, de mettre à jour des statistiques ou d'initialiser un profil.

Le système d'événements de Spring fournit un mécanisme basé sur le pattern **Observer** pour découpler les composants.
Un composant (le *publisher* ou émetteur) publie un événement lorsqu'une action spécifique se produit, et d'autres
composants (les *listeners* ou écouteurs) peuvent réagir à cet événement sans que l'émetteur ait besoin de les connaître
directement.

Ce découplage améliore la maintenabilité, la testabilité et la flexibilité de l'application. Dans un contexte web, cela
permet de séparer la logique principale du contrôleur ou du service des tâches annexes.

## Concepts Fondamentaux: `ApplicationEvent` et `ApplicationEventPublisher`

### `ApplicationEvent`

* **Définition:** Toute action ou état significatif dans l'application peut être représenté par un événement. Dans
  Spring, les événements personnalisés doivent hériter de la classe `ApplicationEvent`.
* **Création d'un Événement Personnalisé :** Pour créer un événement spécifique, on étend `ApplicationEvent`. L'événement
  peut transporter des données pertinentes concernant ce qui s'est passé. Il est recommandé de rendre les objets
  événements **immutables**.

```java
package fr.formation.spring.event;

import org.springframework.context.ApplicationEvent;
// Importe l'entité ou le DTO pertinent si nécessaire
import fr.formation.spring.user.UserAccount; // Supposons une classe UserAccount

/**
 * Événement déclenché lors de la création réussie d'un compte utilisateur.
 * Il hérite de ApplicationEvent et transporte l'objet UserAccount créé.
 */
public class UserCreationEvent extends ApplicationEvent {

    private final UserAccount userAccount; // Données liées à l'événement

    /**
     * Constructeur de l'événement.
     * @param source L'objet sur lequel l'événement s'est initialement produit
     *               (généralement le composant qui publie l'événement).
     * @param userAccount L'objet UserAccount qui vient d'être créé.
     */
    public UserCreationEvent(Object source, UserAccount userAccount) {
        super(source);
        this.userAccount = userAccount;
    }

    /**
     * Getter pour accéder aux données de l'événement 
     * (l'utilisateur créé).
     * @return Le compte utilisateur associé à cet événement.
     */
    public UserAccount getUserAccount() {
        return userAccount;
    }
}
```

### `ApplicationEventPublisher`

* **Définition:** C'est une interface Spring qui permet de publier des événements. Les beans Spring peuvent obtenir une
  instance de `ApplicationEventPublisher` par injection de dépendances.
* **Publication d'un Événement:** Pour publier un événement, on appelle la méthode `publishEvent()` de l'instance
  `ApplicationEventPublisher`.

```java
package fr.formation.spring.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
// Importe l'entité ou le DTO et l'événement
import fr.formation.spring.user.UserAccount;
import fr.formation.spring.user.UserAccountRepository; // Supposons un Repository
import fr.formation.spring.event.UserCreationEvent;

@Service
public class UserAccountService {

    private final ApplicationEventPublisher eventPublisher;
    private final UserAccountRepository userRepository; // Exemple de dépendance

    // Injection de dépendances via le constructeur (bonne pratique)
    @Autowired
    public UserAccountService(
            ApplicationEventPublisher eventPublisher,
            UserAccountRepository userRepository
    ) {
        this.eventPublisher = eventPublisher;
        this.userRepository = userRepository;
    }

    /**
     * Méthode pour enregistrer un nouvel utilisateur.
     * Après sauvegarde, publie un événement UserCreationEvent.
     * @param username Le nom d'utilisateur.
     * @param password Le mot de passe (devrait être hashé !).
     * @param email L'adresse email.
     * @return Le compte utilisateur créé et sauvegardé.
     */
    public UserAccount registerUser(String username, String password, String email) {
        // Logique de création de l'utilisateur
        UserAccount newUser = new UserAccount();
        newUser.setUsername(username);
        // !!! Attention: Hasher le mot de passe en production !!!
        newUser.setPassword(password);
        newUser.setEmail(email);

        // Sauvegarde en base de données (supposée)
        UserAccount savedUser = userRepository.save(newUser);

        // Publication de l'événement APRES la sauvegarde réussie
        // 'this' est la source de l'événement
        UserCreationEvent event = new UserCreationEvent(this, savedUser);
        this.eventPublisher.publishEvent(event);
        System.out.println("Événement UserCreationEvent publié pour: "
                + savedUser.getUsername());

        return savedUser;
    }
}
```

### Écouteurs d'Événements: `@EventListener`

* **Définition:** Pour réagir à un événement, on crée un composant Spring (par exemple, un `@Service` ou `@Component`)
  contenant une méthode annotée avec `@EventListener`. Cette méthode sera automatiquement invoquée lorsque l'événement
  spécifié (ou un de ses sous-types) sera publié.
* **Signature de la Méthode:** La méthode d'écoute doit accepter en paramètre le type d'événement qu'elle souhaite
  écouter.

```java
package fr.formation.spring.listener;

import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
// Importe l'événement spécifique
import fr.formation.spring.event.UserCreationEvent;
// Importe un service (ex: email) si nécessaire
import fr.formation.spring.service.EmailService; // Supposons un EmailService

@Component // Déclare ce bean comme un composant Spring
public class UserRegistrationListener {

    private final EmailService emailService; // Service pour envoyer des emails

    // Injection de dépendances
    public UserRegistrationListener(EmailService emailService) {
        this.emailService = emailService;
    }

    /**
     * Méthode qui écoute les événements de type UserCreationEvent.
     * Elle est déclenchée après la publication de l'événement.
     * @param event L'événement UserCreationEvent contenant les données.
     */
    @EventListener
    public void handleUserCreationEvent(UserCreationEvent event) {
        System.out.println(
                "Listener: Événement UserCreationEvent reçu pour l'utilisateur: "
                        + event.getUserAccount().getUsername()
        );

        // Logique à exécuter en réponse à l'événement
        // Par exemple, envoyer un email de bienvenue
        String recipientEmail = event.getUserAccount().getEmail();
        String subject = "Bienvenue chez Nous !";
        String body = "Bonjour " + event.getUserAccount().getUsername()
                + ", bienvenue sur notre plateforme !";

        try {
            emailService.sendWelcomeEmail(recipientEmail, subject, body);
            System.out.println(
                    "Listener: Email de bienvenue envoyé à " + recipientEmail
            );
        } catch (Exception e) {
            // Gérer les erreurs d'envoi d'email
            System.err.println(
                    "Listener: Erreur lors de l'envoi de l'email à "
                            + recipientEmail + ": " + e.getMessage()
            );
            // Envisager une logique de retry ou de log d'erreur persistante
        }
    }
}
```

---

### Exercice 1: Création et Écoute d'un Événement Simple

**Énoncé:**

1. Créez un événement `ProductViewedEvent` qui se déclenche lorsqu'un utilisateur consulte la page de détail d'un
   produit. Cet événement doit contenir l'ID (`Long productId`) du produit consulté.
2. Créez un service `ProductService` avec une méthode `viewProduct(Long productId)` qui simule la récupération d'un
   produit et publie ensuite l'événement `ProductViewedEvent`.
3. Créez un listener `ProductAnalyticsListener` qui écoute `ProductViewedEvent` et affiche simplement un message dans la
   console indiquant quel produit a été vu (par exemple: "Produit ID: [productId] consulté.").

**Fichiers à créer/modifier:**

* `fr.formation.spring.event.ProductViewedEvent.java`
* `fr.formation.spring.service.ProductService.java`
* `fr.formation.spring.listener.ProductAnalyticsListener.java`

---

#### Correction Exercice 1 {collapsible="true"}

* **`fr.formation.spring.event.ProductViewedEvent.java`**

```java
package fr.formation.spring.event;

import org.springframework.context.ApplicationEvent;

/**
 * Événement indiquant qu'un produit spécifique a été consulté.
 */
public class ProductViewedEvent extends ApplicationEvent {

    private final Long productId; // ID du produit consulté

    /**
     * Constructeur.
     * @param source L'objet source de l'événement.
     * @param productId L'ID du produit qui a été vu.
     */
    public ProductViewedEvent(Object source, Long productId) {
        super(source);
        this.productId = productId;
    }

    /**
     * Récupère l'ID du produit associé à cet événement.
     * @return L'ID du produit.
     */
    public Long getProductId() {
        return productId;
    }
}
```

* **`fr.formation.spring.service.ProductService.java`**

```java
package fr.formation.spring.service;

import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import fr.formation.spring.event.ProductViewedEvent;

@Service
public class ProductService {

    private final ApplicationEventPublisher eventPublisher;

    public ProductService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    /**
     * Simule la consultation d'un produit et publie un événement.
     * @param productId L'ID du produit consulté.
     */
    public void viewProduct(Long productId) {
        // Simule la logique métier (ex: récupérer le produit de la DB)
        System.out.println(
                "Service: Consultation du produit ID: " + productId
        );

        // Publie l'événement après la consultation
        ProductViewedEvent event = new ProductViewedEvent(this, productId);
        this.eventPublisher.publishEvent(event);
        System.out.println(
                "Service: Événement ProductViewedEvent publié pour ID: " + productId
        );
    }
}
```

* **`fr.formation.spring.listener.ProductAnalyticsListener.java`**

```java
package fr.formation.spring.listener;

import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
import fr.formation.spring.event.ProductViewedEvent;

@Component
public class ProductAnalyticsListener {

    /**
     * Écoute l'événement ProductViewedEvent.
     * @param event L'événement contenant l'ID du produit.
     */
    @EventListener
    public void handleProductViewed(ProductViewedEvent event) {
        // Simule une action d'analyse ou de log
        System.out.println(
                "Listener Analytics: Produit ID: " + event.getProductId()
                        + " consulté."
        );
        // Ici pourrait se trouver une logique pour incrémenter un compteur,
        // envoyer des données à un système d'analyse, etc.
    }
}
```

* **Pour tester (dans une classe de test ou un `@PostConstruct`) :**

```java
// ... Dans une classe annotée @Component ou @SpringBootApplication ...
@Autowired
private ProductService productService;

@PostConstruct
public void testProductView() {
    productService.viewProduct(123L);
}
```

---

## Gestion des Événements et Transactions: `@TransactionalEventListener`

### Le Problème

Considérons notre exemple d'inscription utilisateur (`UserCreationEvent`). L'événement est publié *à l'intérieur* de la
méthode `registerUser`. Si cette méthode est transactionnelle (`@Transactional`), la publication peut avoir lieu *avant*
que la transaction ne soit validée (commit).

Si l'envoi de l'email (dans le listener) réussit, mais que la transaction échoue et effectue un rollback *après* la
publication, l'utilisateur recevra un email de bienvenue pour un compte qui n'existe finalement pas en base de données.
C'est une incohérence !

### La Solution: `@TransactionalEventListener`

Spring fournit `@TransactionalEventListener` comme alternative à `@EventListener`. Cette annotation permet de lier
l'exécution de l'écouteur à une phase spécifique de la transaction de l'émetteur.

* **Phases Principales:**
    * `AFTER_COMMIT` (par défaut): L'écouteur n'est exécuté que si la transaction de l'émetteur est validée avec succès.
      C'est le cas d'usage le plus courant pour garantir la cohérence.
    * `AFTER_ROLLBACK`: L'écouteur est exécuté si la transaction échoue et est annulée. Utile pour des actions de
      nettoyage ou de notification d'échec.
    * `AFTER_COMPLETION`: L'écouteur est exécuté après la fin de la transaction, qu'elle soit validée ou annulée (commit
      ou rollback).
    * `BEFORE_COMMIT`: L'écouteur est exécuté juste avant le commit de la transaction. Usage moins fréquent,
      potentiellement pour des validations finales.

### Exemple d'Utilisation

Modifions notre `UserRegistrationListener` pour qu'il n'envoie l'email qu'après le commit de la transaction.

```java
package fr.formation.spring.listener;

// ... autres imports ...

import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase; // Important
import org.springframework.transaction.event.TransactionalEventListener; // Important
import fr.formation.spring.event.UserCreationEvent;
import fr.formation.spring.service.EmailService;

@Component
public class UserRegistrationTransactionalListener { // Renommé pour clarté

    private final EmailService emailService;

    public UserRegistrationTransactionalListener(EmailService emailService) {
        this.emailService = emailService;
    }

    /**
     * Écoute UserCreationEvent, mais seulement APRES le commit
     * de la transaction dans laquelle l'événement a été publié.
     * @param event L'événement contenant les données utilisateur.
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    // Si phase non spécifiée, AFTER_COMMIT est utilisé par défaut
    // @TransactionalEventListener // Équivalent
    public void handleUserCreationCommit(UserCreationEvent event) {
        System.out.println(
                "[Transactional Listener AFTER_COMMIT]: " +
                        "Transaction validée pour user " + event.getUserAccount().getUsername()
                        + ". Envoi de l'email..."
        );

        String recipientEmail = event.getUserAccount().getEmail();
        // ... logique d'envoi d'email ...
        try {
            emailService.sendWelcomeEmail(recipientEmail, "Bienvenue", "...");
            System.out.println(
                    "[Transactional Listener AFTER_COMMIT]: Email envoyé à "
                            + recipientEmail
            );
        } catch (Exception e) {
            System.err.println(
                    "[Transactional Listener AFTER_COMMIT]: Erreur email: "
                            + e.getMessage()
            );
            // Que faire si l'email échoue APRES le commit ?
            // Logguer l'erreur, mettre dans une file d'attente de retry, etc.
        }
    }

    /**
     * Exemple d'écouteur pour le cas du rollback.
     * @param event L'événement.
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleUserCreationRollback(UserCreationEvent event) {
        System.out.println(
                "[Transactional Listener AFTER_ROLLBACK]: " +
                        "Transaction annulée pour user " + event.getUserAccount().getUsername()
                        + ". Aucune action requise ou log spécifique."
        );
        // Utile pour notifier un admin, ou nettoyer des ressources externes
        // qui auraient pu être créées avant le rollback (rare si bien conçu).
    }
}
```

**Important:** Pour que `@TransactionalEventListener` fonctionne, la méthode qui publie l'événement (ici `registerUser`)
doit s'exécuter dans le contexte d'une transaction Spring (par exemple, être annotée avec `@Transactional` ou appelée
par une méthode qui l'est).

```java
package fr.formation.spring.service;

// ... autres imports ...

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional; // Important

@Service
public class UserAccountService {
    // ... constructeur et autres dépendances ...

    @Transactional // Assure que la méthode s'exécute dans une transaction
    public UserAccount registerUser(String username, String password, String email) {
        // ... logique de création et sauvegarde ...
        UserAccount savedUser = userRepository.save(newUser);

        // L'événement est publié DANS la transaction
        UserCreationEvent event = new UserCreationEvent(this, savedUser);
        this.eventPublisher.publishEvent(event);
        System.out.println(
                "Événement UserCreationEvent publié (dans transaction) pour: "
                        + savedUser.getUsername()
        );

        // // Décommenter pour simuler un rollback:
        // if (true) {
        //     throw new RuntimeException("Simulation d'échec après publication !");
        // }

        return savedUser; // Commit implicite à la fin si pas d'exception
    }
}
```

**Cas d'Utilisation Typique:** Envoi de notifications (email, SMS), mise à jour de systèmes externes, invalidation de
cache, déclenchement de tâches asynchrones, *uniquement* si l'opération principale en base de données a réussi.

---

### Exercice 2: Utilisation de `@TransactionalEventListener`

**Énoncé:**
On suppose qu'ajouter un produit au panier (`CartItem`) implique une sauvegarde en base de données et que cette
opération est transactionnelle.

1. Créez un événement `ItemAddedToCartEvent` contenant l'ID de l'utilisateur (`Long userId`) et l'ID du produit (
   `Long productId`).
2. Modifiez (ou créez) un service `ShoppingCartService` avec une méthode
   `@Transactional addItemToCart(Long userId, Long productId)` qui simule l'ajout au panier (par exemple, sauvegarde une
   entité `CartItem`) et publie l'événement `ItemAddedToCartEvent`.
3. Créez un listener `CartUpdateNotifierListener` qui utilise `@TransactionalEventListener` pour écouter
   `ItemAddedToCartEvent` *uniquement après le commit* de la transaction. Ce listener affichera un message dans la
   console indiquant que le panier de l'utilisateur a été mis à jour.
4. (Optionnel) Ajoutez un second listener dans `CartUpdateNotifierListener` qui écoute le même événement mais en phase
   `AFTER_ROLLBACK` et affiche un message différent.

**Fichiers à créer/modifier:**

* `fr.formation.spring.event.ItemAddedToCartEvent.java`
* `fr.formation.spring.service.ShoppingCartService.java` (avec une dépendance simulée vers un `CartItemRepository`)
* `fr.formation.spring.listener.CartUpdateNotifierListener.java`
* (Optionnel) `fr.formation.spring.model.CartItem.java` (simple classe POJO ou Entité JPA)
* (Optionnel) `fr.formation.spring.repository.CartItemRepository.java` (Interface simple)

---

#### Correction Exercice 2 {collapsible="true"}

* **`fr.formation.spring.event.ItemAddedToCartEvent.java`**

```java
package fr.formation.spring.event;

import org.springframework.context.ApplicationEvent;

/**
 * Événement indiquant qu'un article a été ajouté au panier d'un utilisateur.
 */
public class ItemAddedToCartEvent extends ApplicationEvent {

    private final Long userId;
    private final Long productId;

    public ItemAddedToCartEvent(Object source, Long userId, Long productId) {
        super(source);
        this.userId = userId;
        this.productId = productId;
    }

    public Long getUserId() {
        return userId;
    }

    public Long getProductId() {
        return productId;
    }
}
```

* **(Optionnel) `fr.formation.spring.model.CartItem.java`**

```java
package fr.formation.spring.model;

// Simulation simple, pourrait être une @Entity JPA
public class CartItem {
    private Long id;
    private Long userId;
    private Long productId;
    private int quantity;
    // Getters, Setters...
}
```

* **(Optionnel) `fr.formation.spring.repository.CartItemRepository.java`**

```java
package fr.formation.spring.repository;

import org.springframework.stereotype.Repository;
import fr.formation.spring.model.CartItem;

// Simulation d'un repository
@Repository
public interface CartItemRepository {
    CartItem save(CartItem item); // Méthode simulée
}

// Implémentation factice pour l'exemple si pas de JPA/JDBC configuré
@Repository
class MockCartItemRepositoryImpl implements CartItemRepository {
    @Override
    public CartItem save(CartItem item) {
        System.out.println("Simulating save of CartItem for user "
                + item.getUserId() + ", product " + item.getProductId());
        // Simule l'attribution d'un ID
        item.setId((long) (Math.random() * 1000));
        return item;
    }
}
```

* **`fr.formation.spring.service.ShoppingCartService.java`**

```java
package fr.formation.spring.service;

import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import fr.formation.spring.event.ItemAddedToCartEvent;
import fr.formation.spring.model.CartItem; // Si défini
import fr.formation.spring.repository.CartItemRepository; // Si défini

@Service
public class ShoppingCartService {

    private final ApplicationEventPublisher eventPublisher;
    private final CartItemRepository cartItemRepository; // Injection repo

    public ShoppingCartService(ApplicationEventPublisher eventPublisher,
                               CartItemRepository cartItemRepository) {
        this.eventPublisher = eventPublisher;
        this.cartItemRepository = cartItemRepository;
    }

    /**
     * Ajoute un article au panier (transactionnel).
     * Publie un événement ItemAddedToCartEvent si réussi.
     * @param userId ID de l'utilisateur.
     * @param productId ID du produit.
     * @return Le CartItem sauvegardé (ou simulé).
     */
    @Transactional // Important pour @TransactionalEventListener
    public CartItem addItemToCart(Long userId, Long productId) {
        System.out.println("Service: Ajout du produit " + productId
                + " au panier de l'utilisateur " + userId);

        // Simule la création/mise à jour de l'item dans le panier
        CartItem item = new CartItem(); // Créer un objet CartItem
        item.setUserId(userId);
        item.setProductId(productId);
        item.setQuantity(1); // ou incrémenter si existant

        CartItem savedItem = cartItemRepository.save(item); // Sauvegarde DB

        // Publie l'événement DANS la transaction
        ItemAddedToCartEvent event =
                new ItemAddedToCartEvent(this, userId, productId);
        this.eventPublisher.publishEvent(event);
        System.out.println("Service: Événement ItemAddedToCartEvent publié.");

        // // Décommenter pour tester le rollback:
        // if (productId == 999L) { // Condition pour simuler un échec
        //     throw new RuntimeException("Échec simulé de l'ajout au panier !");
        // }

        return savedItem; // Commit implicite ici si pas d'exception
    }
}
```

* **`fr.formation.spring.listener.CartUpdateNotifierListener.java`**

```java
package fr.formation.spring.listener;

import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;
import fr.formation.spring.event.ItemAddedToCartEvent;

@Component
public class CartUpdateNotifierListener {

    /**
     * Notifie après commit réussi de l'ajout au panier.
     * @param event L'événement contenant les détails.
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleCartUpdateCommit(ItemAddedToCartEvent event) {
        System.out.println(
                "[Notifier Listener AFTER_COMMIT]: Le panier de l'utilisateur "
                        + event.getUserId() + " a été mis à jour avec le produit "
                        + event.getProductId() + ". Transaction validée."
        );
        // Ici, on pourrait rafraîchir une partie de l'UI via WebSockets,
        // ou notifier un autre système.
    }

    /**
     * (Optionnel) Gère le cas où la transaction d'ajout a échoué.
     * @param event L'événement.
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleCartUpdateRollback(ItemAddedToCartEvent event) {
        System.out.println(
                "[Notifier Listener AFTER_ROLLBACK]: Échec de l'ajout du produit "
                        + event.getProductId() + " pour l'utilisateur "
                        + event.getUserId() + ". Transaction annulée."
        );
        // Logguer l'échec, notifier l'admin si nécessaire.
    }
}
```

* **Pour tester (simuler un contrôleur web ou autre service):**

```java
// ... Dans une classe de test ou composant ...
@Autowired
private ShoppingCartService cartService;

public void simulateAddToCart() {
    try {
        cartService.addItemToCart(1L, 101L); // Cas succès
    } catch (Exception e) {
        System.err.println("Erreur lors de l'ajout : " + e.getMessage());
    }
    try {
        // Pour tester le rollback, décommenter le throw dans le service
        // et appeler avec productId = 999L
        // cartService.addItemToCart(2L, 999L);
    } catch (Exception e) {
        System.err.println("Erreur attendue lors de l'ajout : " + e.getMessage());
    }
}
```

---

## Écouteurs d'Entités JPA: `@EntityListeners`

### Contexte

Parfois, on souhaite réagir spécifiquement aux événements du cycle de vie d'une entité JPA (création, mise à jour,
suppression), par exemple pour l'audit (qui a créé/modifié quoi et quand), ou pour mettre à jour des champs dérivés. JPA
fournit des annotations de callback de cycle de vie (`@PrePersist`, `@PostPersist`, `@PreUpdate`, `@PostUpdate`,
`@PreRemove`, `@PostRemove`, `@PostLoad`).

On peut placer ces annotations directement dans l'entité ou, pour une meilleure séparation des préoccupations (surtout
si la logique est complexe ou réutilisée), dans une classe externe appelée *Entity Listener*.

### ` @EntityListeners`

L'annotation `@EntityListeners` s'applique sur une classe d'entité JPA et spécifie une ou plusieurs classes d'écouteurs
qui contiennent les méthodes de callback.

### Création d'un Entity Listener

Un Entity Listener est une simple classe (pas nécessairement un bean Spring par défaut, mais peut l'être) contenant des
méthodes annotées avec les annotations de callback JPA. Ces méthodes reçoivent l'entité concernée en paramètre.

```java
package fr.formation.spring.jpa.listener;

import java.time.LocalDateTime;
import javax.persistence.PrePersist;
import javax.persistence.PreUpdate;
// Importe une interface/classe de base pour les entités auditables
import fr.formation.spring.jpa.model.Auditable;

/**
 * Écouteur d'entité JPA pour gérer automatiquement les champs d'audit
 * (date de création et de dernière modification).
 */
public class AuditListener {

    /**
     * Méthode exécutée AVANT la persistance (création) d'une entité.
     * Initialise les dates de création et de mise à jour.
     * @param entity L'objet entité sur le point d'être persisté.
     *               Doit implémenter l'interface Auditable.
     */
    @PrePersist
    public void setCreationTimestamp(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditableEntity = (Auditable) entity;
            LocalDateTime now = LocalDateTime.now();
            if (auditableEntity.getCreatedAt() == null) {
                auditableEntity.setCreatedAt(now);
            }
            // La date de màj est aussi mise à jour lors de la création
            auditableEntity.setUpdatedAt(now);
            System.out.println("AuditListener @PrePersist: Set timestamps for "
                    + entity.getClass().getSimpleName());
        }
    }

    /**
     * Méthode exécutée AVANT la mise à jour d'une entité existante.
     * Met à jour la date de dernière modification.
     * @param entity L'objet entité sur le point d'être mis à jour.
     *               Doit implémenter l'interface Auditable.
     */
    @PreUpdate
    public void setUpdateTimestamp(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditableEntity = (Auditable) entity;
            auditableEntity.setUpdatedAt(LocalDateTime.now());
            System.out.println("AuditListener @PreUpdate: Set update timestamp for "
                    + entity.getClass().getSimpleName());
        }
    }
}

// Interface commune pour les entités nécessitant un audit de date
package fr.formation.spring.jpa.model;

import java.time.LocalDateTime;

public interface Auditable {
    LocalDateTime getCreatedAt();

    void setCreatedAt(LocalDateTime createdAt);

    LocalDateTime getUpdatedAt();

    void setUpdatedAt(LocalDateTime updatedAt);
}
```

### Application à une Entité

Il suffit ensuite d'annoter l'entité JPA avec `@EntityListeners`.

```java
package fr.formation.spring.jpa.model;

import java.time.LocalDateTime;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.EntityListeners; // Important
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
// Importe l'interface et le listener
import fr.formation.spring.jpa.listener.AuditListener;

@Entity
@EntityListeners(AuditListener.class) // Applique le listener à cette entité
public class Order implements Auditable { // Implémente l'interface Auditable

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String orderNumber;
    // ... autres champs de la commande ...

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt; // Champ pour la date de création

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt; // Champ pour la date de mise à jour

    // Implémentation des méthodes de l'interface Auditable
    @Override
    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    @Override
    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }

    @Override
    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }

    @Override
    public void setUpdatedAt(LocalDateTime updatedAt) {
        this.updatedAt = updatedAt;
    }

    // Getters, Setters pour les autres champs (id, orderNumber...)
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getOrderNumber() {
        return orderNumber;
    }

    public void setOrderNumber(String orderNumber) {
        this.orderNumber = orderNumber;
    }
}
```

Maintenant, chaque fois qu'une entité `Order` sera créée (`persist`) ou mise à jour (`merge`), les méthodes
`@PrePersist` et `@PreUpdate` de `AuditListener` seront automatiquement appelées par JPA pour remplir les champs
`createdAt` et `updatedAt`.

**Lien avec les Événements Spring:**
Bien que les `EntityListeners` soient un mécanisme JPA, ils peuvent être utilisés pour *publier* des `ApplicationEvent`
Spring si nécessaire. Par exemple, un `@PostPersist` pourrait utiliser un `ApplicationEventPublisher` (injecté si
l'EntityListener est géré par Spring, voir ci-dessous) pour signaler la création d'une entité à d'autres parties de
l'application.

**Injection de Dépendances dans les Entity Listeners:**
Par défaut, les Entity Listeners sont instanciés directement par JPA et ne sont pas des beans Spring. Si un Entity
Listener a besoin de dépendances Spring (comme `ApplicationEventPublisher`), il faut configurer JPA pour qu'il utilise
le contexte Spring pour obtenir les instances de listeners. Cela se fait souvent via la configuration de
`LocalContainerEntityManagerFactoryBean` ou via des propriétés spécifiques au fournisseur JPA (Hibernate, EclipseLink).
Une alternative plus simple est parfois d'utiliser un bean Spring statique accessible depuis le listener. Spring Data
JPA fournit également des facilités pour l'audit (`@EnableJpaAuditing`, `@CreatedDate`, `@LastModifiedDate`) qui sont
souvent préférables pour ce cas d'usage précis.

---

### Exercice 3: Mise en place d'un Audit Simple avec `@EntityListeners`

**Énoncé:**

1. Créez une entité JPA simple `BlogPost` avec au moins un champ `title` (String) et un champ `content` (String).
2. Ajoutez les champs `creationDate` (LocalDateTime) et `lastUpdateDate` (LocalDateTime) à l'entité `BlogPost`.
3. Créez une interface `Timestamped` avec les méthodes `getCreationDate()`, `setCreationDate()`, `getLastUpdateDate()`,
   `setLastUpdateDate()`. Faites implémenter cette interface par `BlogPost`.
4. Créez un `EntityListener` nommé `TimestampListener` avec des méthodes `@PrePersist` et `@PreUpdate` pour
   initialiser/mettre à jour ces dates, similaire à l'exemple `AuditListener`.
5. Annotez l'entité `BlogPost` avec `@EntityListeners(TimestampListener.class)`.
6. (Simulation) Dans un service ou un test, créez une instance de `BlogPost`, sauvegardez-la (simulez avec un
   `EntityManager` ou un `Repository`), modifiez-la, puis sauvegardez à nouveau. Vérifiez (par affichage console dans le
   listener ou en récupérant l'entité) que les dates ont été correctement définies.

**Fichiers à créer/modifier:**

* `fr.formation.spring.jpa.model.BlogPost.java`
* `fr.formation.spring.jpa.model.Timestamped.java` (Interface)
* `fr.formation.spring.jpa.listener.TimestampListener.java`
* (Optionnel) Un service ou une classe de test pour simuler la persistance.

---

#### Correction Exercice 3 {collapsible="true"}

* **`fr.formation.spring.jpa.model.Timestamped.java`**

```java
package fr.formation.spring.jpa.model;

import java.time.LocalDateTime;

/**
 * Interface pour les entités qui doivent avoir des timestamps de création/màj.
 */
public interface Timestamped {
    LocalDateTime getCreationDate();

    void setCreationDate(LocalDateTime creationDate);

    LocalDateTime getLastUpdateDate();

    void setLastUpdateDate(LocalDateTime lastUpdateDate);
}
```

* **`fr.formation.spring.jpa.model.BlogPost.java`**

```java
package fr.formation.spring.jpa.model;

import java.time.LocalDateTime;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.EntityListeners;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Lob; // Pour les champs texte longs

import fr.formation.spring.jpa.listener.TimestampListener; // Import listener

@Entity
@EntityListeners(TimestampListener.class) // Applique le listener
public class BlogPost implements Timestamped { // Implémente l'interface

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Lob // Indique un champ potentiellement grand (CLOB/TEXT)
    private String content;

    @Column(name = "creation_date", nullable = false, updatable = false)
    private LocalDateTime creationDate;

    @Column(name = "last_update_date", nullable = false)
    private LocalDateTime lastUpdateDate;

    // Implémentation interface Timestamped
    @Override
    public LocalDateTime getCreationDate() {
        return creationDate;
    }

    @Override
    public void setCreationDate(LocalDateTime date) {
        this.creationDate = date;
    }

    @Override
    public LocalDateTime getLastUpdateDate() {
        return lastUpdateDate;
    }

    @Override
    public void setLastUpdateDate(LocalDateTime date) {
        this.lastUpdateDate = date;
    }

    // Getters & Setters pour id, title, content
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    @Override
    public String toString() {
        return "BlogPost [id=" + id + ", title=" + title
                + ", creationDate=" + creationDate
                + ", lastUpdateDate=" + lastUpdateDate + "]";
    }
}
```

* **`fr.formation.spring.jpa.listener.TimestampListener.java`**

```java
package fr.formation.spring.jpa.listener;

import java.time.LocalDateTime;
import javax.persistence.PrePersist;
import javax.persistence.PreUpdate;

import fr.formation.spring.jpa.model.Timestamped; // Import interface

/**
 * Entity Listener pour gérer les timestamps de création et de mise à jour.
 */
public class TimestampListener {

    @PrePersist
    public void onPrePersist(Object entity) {
        if (entity instanceof Timestamped) {
            Timestamped timestampedEntity = (Timestamped) entity;
            LocalDateTime now = LocalDateTime.now();
            if (timestampedEntity.getCreationDate() == null) {
                timestampedEntity.setCreationDate(now);
            }
            timestampedEntity.setLastUpdateDate(now); // Aussi lors création
            System.out.println(
                    "TimestampListener @PrePersist: Set timestamps on "
                            + entity.getClass().getSimpleName()
            );
        }
    }

    @PreUpdate
    public void onPreUpdate(Object entity) {
        if (entity instanceof Timestamped) {
            Timestamped timestampedEntity = (Timestamped) entity;
            timestampedEntity.setLastUpdateDate(LocalDateTime.now());
            System.out.println(
                    "TimestampListener @PreUpdate: Set lastUpdateDate on "
                            + entity.getClass().getSimpleName()
            );
        }
    }
}
```

* **Simulation de persistance (Exemple avec Spring Data JPA Repository):**

```java
package fr.formation.spring.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import fr.formation.spring.jpa.model.BlogPost;

// Suppose que Spring Data JPA est configuré
public interface BlogPostRepository extends JpaRepository<BlogPost, Long> {
}

// --- Dans un service ou une classe de test ---
package fr.formation.spring.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import fr.formation.spring.jpa.model.BlogPost;
import fr.formation.spring.repository.BlogPostRepository;

@Service
public class BlogService {

    @Autowired
    private BlogPostRepository blogPostRepository;

    @Transactional // Important pour la gestion par JPA/Hibernate
    public void demonstrateTimestamps() {
        System.out.println("--- Création du Post ---");
        BlogPost post = new BlogPost();
        post.setTitle("Mon Premier Article");
        post.setContent("Contenu de l'article...");

        // Sauvegarde initiale (déclenche @PrePersist)
        BlogPost savedPost = blogPostRepository.save(post);
        System.out.println("Post sauvegardé: " + savedPost);

        // Pause pour voir une différence dans les dates
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
        }

        System.out.println("\n--- Mise à jour du Post ---");
        // Récupération (optionnel, save fait aussi un merge si ID existe)
        BlogPost postToUpdate = blogPostRepository.findById(savedPost.getId())
                .orElseThrow();
        postToUpdate.setContent("Contenu mis à jour !");

        // Sauvegarde (déclenche @PreUpdate)
        BlogPost updatedPost = blogPostRepository.save(postToUpdate);
        System.out.println("Post mis à jour: " + updatedPost);
    }
}

// --- Pour exécuter (ex: dans la classe principale ou un test) ---
// @Autowired BlogService blogService;
// blogService.demonstrateTimestamps();
```

L'exécution de `demonstrateTimestamps` devrait montrer dans la console les messages du `TimestampListener` et les objets
`BlogPost` avec les dates `creationDate` et `lastUpdateDate` correctement renseignées et mises à jour.




## Cas d'Utilisation Typiques dans une Application Web

Les événements sont particulièrement utiles dans les applications web pour :

1. **Notifications Utilisateur Post-Action:**
    * Envoyer un email de confirmation après inscription (`UserCreationEvent`).
    * Notifier l'utilisateur qu'une commande a été expédiée (`OrderShippedEvent`).
    * Signaler la réussite d'une opération longue (import de données, génération de rapport).
2. **Découplage des Tâches de Fond:**
    * Après la mise à jour d'un produit, déclencher une réindexation dans un moteur de recherche (
      `ProductUpdatedEvent`).
    * Invalider des caches spécifiques lorsqu'une donnée est modifiée (`CacheInvalidationEvent`).
    * Planifier des tâches de nettoyage ou de maintenance suite à certaines actions.
3. **Audit et Journalisation:**
    * Enregistrer des événements de sécurité importants (connexion réussie/échouée, modification de droits) dans un
      journal d'audit (`SecurityAuditEvent`).
    * Tracer des parcours utilisateurs complexes en enregistrant des étapes clés sous forme d'événements.
4. **Statistiques et Métriques:**
    * Incrémenter des compteurs pour des actions spécifiques (produit ajouté au panier, article publié) en écoutant les
      événements correspondants.
5. **Intégration Légère (Interne):**
    * Permettre à différents modules de l'application de réagir aux actions des autres sans dépendances directes. Par
      exemple, un module de gestion de stock réagit à un `OrderPlacedEvent` du module de commandes.

---

## Bonnes Pratiques et Conseils

1. **Nommage Clair:** Donnez des noms explicites à vos événements (souvent un nom au participe passé : `OrderCreated`,
   `UserLoggedIn`).
2. **Granularité:** Créez des événements spécifiques plutôt qu'un événement générique avec un type.
   `UserRegisteredEvent` est mieux que `SystemEvent(type='USER_REGISTRATION')`.
3. **Immutabilité:** Rendez les objets événements immutables (champs `final`, pas de setters). Cela évite qu'un listener
   ne modifie l'état de l'événement pour les suivants et améliore la sécurité des threads.
4. **Données Pertinentes:** Incluez uniquement les données nécessaires dans l'événement. Si le listener a besoin de plus
   d'informations, il peut utiliser un ID contenu dans l'événement pour les récupérer via un service. Évitez de mettre
   des entités JPA complètes si seules quelques informations sont utiles, préférez des DTOs ou des IDs.
5. **Utilisez `@TransactionalEventListener`:** Pour toute action de listener qui dépend de la réussite d'une
   transaction (notifications, mises à jour externes), privilégiez
   `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`.
6. **Gestion des Erreurs dans les Listeners:** Un listener qui échoue peut (s'il est synchrone) faire échouer
   l'opération principale ou (s'il est asynchrone ou transactionnel `AFTER_COMMIT`) échouer silencieusement ou logguer
   une erreur. Prévoyez une gestion robuste des erreurs dans les listeners (try-catch, logging, mécanismes de retry si
   pertinent).
7. **Listeners Asynchrones (`@Async`):** Utilisez-les judicieusement pour les tâches longues qui ne doivent pas bloquer
   le thread principal. N'oubliez pas `@EnableAsync`. Soyez conscient de la gestion des transactions dans les threads
   séparés.
8. **Ordre des Listeners (`@Order`):** Si plusieurs listeners écoutent le même événement et que leur ordre d'exécution
   est important (rarement une bonne idée, cela réintroduit un couplage), vous pouvez utiliser l'annotation `@Order` sur
   les méthodes de listener.
9. **Testabilité:** Le découplage facilite les tests. Testez les émetteurs (vérifiez qu'ils publient le bon événement)
   et les listeners (testez leur logique en leur passant un événement manuellement) séparément. Utilisez les outils de
   test Spring pour simuler la publication/réception.
10. **Évitez les Dépendances Cycliques:** Ne créez pas de situations où un listener publie un événement qui déclenche un
    autre listener, qui à son tour publie l'événement initial (ou un autre qui revient au premier).

---

## Conclusion

Le système d'événements de Spring Boot, combiné aux `@TransactionalEventListener` et aux `@EntityListeners` de JPA,
offre des outils puissants pour créer des applications web découplées, maintenables et robustes. En comprenant quand et
comment utiliser ces mécanismes, les développeurs peuvent séparer efficacement les préoccupations, gérer la cohérence
transactionnelle des actions secondaires et réagir aux changements du cycle de vie des données, améliorant ainsi la
qualité globale du code. L'utilisation judicieuse des événements synchrones, asynchrones et transactionnels permet
d'adapter le comportement de l'application aux besoins spécifiques de chaque cas d'usage.