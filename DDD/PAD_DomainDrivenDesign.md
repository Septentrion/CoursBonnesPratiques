# Jour 1 — Après-midi : Domain-Driven Design & Event Storming

**Durée totale :** 3 h 30 (deux sessions séparées par une pause de 15 minutes)  
**Public :** Etudiants en informatique, connaissance préalable des design patterns,
des principes SOLID et de la matinée (Clean Code, refactoring)  
**Langages illustrés :** PHP 8.x et Python 3.10+

---

## Avant-propos : pourquoi le DDD ?

La matinée a abordé la qualité du code à un niveau microscopique : nommage,
fonctions, smells. Elle a montré que le code doit parler un langage humain
compréhensible. L'après-midi pose la question du *quel* langage : celui de la
machine, ou celui du domaine métier ?

Considérons un système de réservation de billets d'avion. Une approche CRUD
classique modélise : `INSERT INTO reservations`, `UPDATE seats SET available = 0`.
Une approche orientée domaine modélise : `Booking.confirm()`,
`Flight.allocateSeat(passenger)`, `BoardingPass.issue()`. La différence n'est
pas esthétique — elle détermine si le code peut évoluer avec les règles métier
sans constamment surprendre les développeurs qui le lisent.

Eric Evans a formalisé cette intuition dans *Domain-Driven Design* (2003) :

> « Le logiciel doit incarner une riche connaissance du domaine. »

Le DDD est à la fois une philosophie de conception (comment penser un système)
et un ensemble de patterns tactiques (comment le coder). Ces deux niveaux sont
indissociables : les patterns sans la philosophie donnent du sur-ingénierie ; la
philosophie sans les patterns reste abstraite.

---

## Session 3 — DDD : fondamentaux

### 3.1 Ubiquitous Language — le langage partagé

#### Définition

L'Ubiquitous Language (langage omniprésent) est le vocabulaire commun partagé
par les développeurs et les experts métier d'un même contexte. Ce vocabulaire
doit être :

- utilisé à l'oral dans toutes les discussions ;
- utilisé dans les spécifications et les tickets ;
- **reflété directement dans le code** — noms de classes, méthodes, variables.

Quand un développeur traduit mentalement « l'expert dit *expédier une commande*,
moi j'appelle ça *updateOrderStatus* », il crée une couche de friction permanente.
Chaque aller-retour entre le langage du code et le langage du métier est une source
d'incompréhension et de bugs.

#### Exemple concret

Un expert comptable parle de *facture proforma*, de *note de crédit*, d'*échéance*,
de *rapprochement bancaire*. Si le code contient `Invoice`, `CreditNote`,
`DueDate`, `BankReconciliation`, n'importe quel développeur rejoignant l'équipe
peut comprendre le domaine en lisant le code. Si le code contient `Doc`, `Entry`,
`Date1`, `Reconcile`, il faut demander à l'expert ce que signifie chaque terme.

```python
# Mauvais : langage technique, découplé du métier
class DataProcessor:
    def process(self, data: dict) -> dict:
        if data["type"] == 1:
            data["status"] = 3
            data["val"] = data["val"] * 0.8
        return data

# Bon : Ubiquitous Language — un expert comptable reconnaît ces termes
class Invoice:
    def apply_early_payment_discount(self) -> None:
        if self.payment_terms == PaymentTerms.NET_30_WITH_DISCOUNT:
            self.total_due = self.total_due * Decimal("0.80")
            self.status    = InvoiceStatus.DISCOUNT_APPLIED
```

#### Comment construire l'Ubiquitous Language

1. **Ateliers de modélisation conjoints** avec les experts métier (l'Event Storming,
   que vous pratiquerez dans la session 4, est l'un des outils les plus efficaces).
2. **Glossaire vivant** : un document maintenu à jour, visible de tous, qui liste
   chaque terme avec sa définition précise dans le contexte du projet.
3. **Revue du langage dans le code** : lors des code reviews, signaler les noms
   qui s'éloignent du glossaire.
4. **Refus des synonymes** : `Order` et `Purchase` ne doivent pas désigner la même
   chose dans le même contexte. Si les experts métier utilisent les deux, c'est le
   signal soit d'un sous-domaine différent, soit d'une ambiguïté à clarifier.

---

### 3.2 Blocs de construction tactiques

Les patterns tactiques du DDD définissent comment modéliser les concepts du domaine
en code. Ils répondent à une seule question : *comment représenter les règles métier
dans des structures orientées objet ?*

---

#### Entity (Entité)

Une entité est un objet dont l'identité est ce qui importe, indépendamment de ses
attributs. Deux entités avec exactement les mêmes attributs sont quand même des
objets distincts si leur identifiant diffère.

**Caractéristiques :**
- possède un identifiant stable qui traverse son cycle de vie ;
- mutable : ses attributs peuvent changer, mais son identité reste la même ;
- l'égalité est définie par l'identifiant, pas par les attributs.

```php
<?php

declare(strict_types=1);

final class CustomerId
{
    public function __construct(public readonly string $value)
    {
        if (empty(trim($value))) {
            throw new \InvalidArgumentException("L'identifiant client ne peut pas être vide.");
        }
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string { return $this->value; }
}

class Customer
{
    private string $name;
    private Email  $email;

    public function __construct(
        private readonly CustomerId $id,
        string $name,
        Email  $email,
    ) {
        $this->name  = $name;
        $this->email = $email;
    }

    public function rename(string $newName): void
    {
        if (empty(trim($newName))) {
            throw new \DomainException("Le nom ne peut pas être vide.");
        }
        $this->name = $newName;
    }

    public function changeEmail(Email $newEmail): void
    {
        $this->email = $newEmail;
    }

    public function equals(self $other): bool
    {
        return $this->id->equals($other->id);
    }

    public function id(): CustomerId { return $this->id; }
    public function name(): string   { return $this->name; }
    public function email(): Email   { return $this->email; }
}

// Deux clients avec le même nom sont distincts si leurs id diffèrent.
// Deux clients avec le même id sont le même client même si leur nom a changé.
```

---

#### Value Object (Objet Valeur)

Un Value Object représente un concept du domaine défini entièrement par ses
attributs. Deux Value Objects avec les mêmes attributs sont interchangeables —
ils sont *égaux*. Il n'y a pas de notion d'identité.

**Caractéristiques :**
- immutable : ses attributs ne changent jamais après la création ;
- l'égalité est définie par les attributs ;
- sans identifiant ;
- auto-validant : les invariants sont vérifiés dans le constructeur.

```python
from __future__ import annotations
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class Money:
    """
    Représente une somme d'argent dans une devise donnée.
    Immutable : toute opération retourne un nouveau Money.
    """
    amount:   Decimal
    currency: str

    def __post_init__(self) -> None:
        if self.amount < Decimal("0"):
            raise ValueError(f"Un montant ne peut pas être négatif : {self.amount}")
        if len(self.currency) != 3:
            raise ValueError(f"Code devise invalide (ISO 4217 attendu) : {self.currency}")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(
                f"Impossible d'additionner {self.currency} et {other.currency}."
            )
        return Money(self.amount + other.amount, self.currency)

    def multiply(self, factor: Decimal) -> "Money":
        return Money((self.amount * factor).quantize(Decimal("0.01")), self.currency)

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"


@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        import re
        if not re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", self.value):
            raise ValueError(f"Adresse e-mail invalide : {self.value}")

    def domain(self) -> str:
        return self.value.split("@")[1]


@dataclass(frozen=True)
class DateRange:
    start: "datetime.date"
    end:   "datetime.date"

    def __post_init__(self) -> None:
        import datetime
        if self.end < self.start:
            raise ValueError("La date de fin doit être postérieure à la date de début.")

    def duration_days(self) -> int:
        return (self.end - self.start).days

    def overlaps(self, other: "DateRange") -> bool:
        return self.start <= other.end and self.end >= other.start


# Preuve d'immutabilité et d'égalité structurelle
price1 = Money(Decimal("100.00"), "EUR")
price2 = Money(Decimal("100.00"), "EUR")
assert price1 == price2   # True : même valeur, mêmes objets

tax    = price1.multiply(Decimal("0.20"))
total  = price1.add(tax)
assert str(total) == "120.00 EUR"
```

**Quand utiliser un Value Object plutôt qu'un type primitif ?**

Dès qu'un concept a des règles de validation, des opérations propres ou un sens
métier fort. `Email` n'est pas une `str` : une `str` peut contenir « bonjour »,
une `Email` ne peut pas. `Money` n'est pas un `float` : deux `float` ne s'additionnent
pas en vérifiant qu'ils ont la même devise.

---

#### Aggregate (Agrégat)

L'Aggregate est le pattern le plus important et le plus mal compris du DDD
tactique. Il répond à une question précise : *quelle est la frontière de cohérence
transactionnelle dans le domaine ?*

**Définition :** un Aggregate est un groupe d'entités et de Value Objects traité
comme une unité de cohérence. L'une d'entre elles est désignée comme **Aggregate
Root** (racine de l'agrégat). Toute modification de l'agrégat passe par la racine ;
rien ne peut modifier les entités internes en contournant la racine.

**Règles de l'Aggregate (Evans, Vaughn Vernon) :**

1. La racine est la seule référence publiée à l'extérieur de l'agrégat.
2. Les entités internes ne sont pas accessibles directement depuis l'extérieur.
3. Un seul agrégat est modifié par transaction.
4. Les références entre agrégats se font par identifiant, jamais par objet.
5. La cohérence entre agrégats est une conséquence (*eventual consistency*), pas immédiate.

```php
<?php

declare(strict_types=1);

// ── Order (Aggregate Root) ────────────────────────────────────────────────────

final class OrderId
{
    public function __construct(public readonly string $value) {}
}

final class OrderLine       // Entité interne à l'agrégat — non exposée directement
{
    public function __construct(
        private readonly ProductId $productId,
        private readonly int       $quantity,
        private readonly Money     $unitPrice,
    ) {}

    public function lineTotal(): Money
    {
        return $this->unitPrice->multiply($this->quantity);
    }

    public function productId(): ProductId { return $this->productId; }
    public function quantity(): int        { return $this->quantity; }
}

class Order                 // Aggregate Root
{
    private OrderStatus      $status;

    /** @var OrderLine[] */
    private array $lines = [];

    /** @var DomainEvent[] */
    private array $domainEvents = [];

    public function __construct(
        private readonly OrderId    $id,
        private readonly CustomerId $customerId,  // référence par id, pas par objet
    ) {
        $this->status = OrderStatus::DRAFT;
    }

    // ── Comportements métier (écrits dans le langage du domaine) ─────────────

    public function addLine(ProductId $productId, int $quantity, Money $unitPrice): void
    {
        $this->ensureNotConfirmed();

        foreach ($this->lines as $line) {
            if ($line->productId()->equals($productId)) {
                throw new \DomainException(
                    "Le produit {$productId->value} est déjà dans la commande. Modifiez la quantité."
                );
            }
        }

        if ($quantity <= 0) {
            throw new \DomainException("La quantité doit être strictement positive.");
        }

        $this->lines[] = new OrderLine($productId, $quantity, $unitPrice);
    }

    public function confirm(): void
    {
        $this->ensureNotConfirmed();
        if (empty($this->lines)) {
            throw new \DomainException("Impossible de confirmer une commande sans lignes.");
        }

        $this->status = OrderStatus::CONFIRMED;
        $this->recordEvent(new OrderConfirmed($this->id, $this->customerId, new \DateTimeImmutable()));
    }

    public function cancel(string $reason): void
    {
        if ($this->status === OrderStatus::SHIPPED) {
            throw new \DomainException("Une commande expédiée ne peut pas être annulée.");
        }
        $this->status = OrderStatus::CANCELLED;
        $this->recordEvent(new OrderCancelled($this->id, $reason, new \DateTimeImmutable()));
    }

    // ── Invariants privés ─────────────────────────────────────────────────────

    private function ensureNotConfirmed(): void
    {
        if ($this->status !== OrderStatus::DRAFT) {
            throw new \DomainException(
                "La commande ne peut être modifiée qu'à l'état brouillon."
            );
        }
    }

    private function recordEvent(DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }

    // ── Accesseurs en lecture (pas de setters publics) ────────────────────────

    public function id(): OrderId          { return $this->id; }
    public function status(): OrderStatus  { return $this->status; }

    public function total(): Money
    {
        return array_reduce(
            $this->lines,
            fn(Money $carry, OrderLine $line) => $carry->add($line->lineTotal()),
            new Money(0, 'EUR'),
        );
    }

    /** @return DomainEvent[] */
    public function pullDomainEvents(): array
    {
        $events            = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

**Taille des agrégats :** Vaughn Vernon recommande de garder les agrégats petits.
Un agrégat de vingt entités est un signal que les frontières sont mal tracées.
La règle empirique : si deux concepts peuvent changer de façon indépendante, ils
appartiennent probablement à deux agrégats distincts.

---

#### Repository (Dépôt)

Le Repository abstrait la persistence d'un agrégat. Il donne l'illusion que les
agrégats vivent en mémoire, sans jamais exposer la technologie de stockage sous-jacente.

**Règles :**
- Un Repository par Aggregate Root (pas un Repository par entité).
- L'interface est définie dans la couche domaine ; l'implémentation dans la couche
  infrastructure.
- Les méthodes parlent le langage du domaine : `findByCustomer()`, pas
  `selectWhereCustomerId()`.

```python
from abc import ABC, abstractmethod


# Interface dans la couche domaine
class OrderRepository(ABC):

    @abstractmethod
    def find_by_id(self, order_id: OrderId) -> Order | None:
        ...

    @abstractmethod
    def find_by_customer(self, customer_id: CustomerId) -> list[Order]:
        ...

    @abstractmethod
    def find_pending_since(self, since: "datetime.datetime") -> list[Order]:
        ...

    @abstractmethod
    def save(self, order: Order) -> None:
        ...

    @abstractmethod
    def remove(self, order_id: OrderId) -> None:
        ...


# Implémentation dans la couche infrastructure (invisible depuis le domaine)
class SqlAlchemyOrderRepository(OrderRepository):
    def __init__(self, session):
        self._session = session

    def find_by_id(self, order_id: OrderId) -> Order | None:
        row = self._session.query(Orderorm).filter_by(id=order_id.value).first()
        return self._to_domain(row) if row else None

    def save(self, order: Order) -> None:
        orm_obj = self._to_orm(order)
        self._session.merge(orm_obj)
        # Publication des Domain Events après le flush
        for event in order.pull_domain_events():
            self._event_bus.publish(event)

    # ... autres méthodes
```

**Lien avec le DIP :** le Repository est l'application la plus directe du
Dependency Inversion Principle dans le DDD. La couche domaine définit l'interface
(l'abstraction) ; la couche infrastructure fournit l'implémentation concrète.
La couche domaine ne dépend *jamais* de SQLAlchemy, de PDO, de Redis ou de tout
autre détail de persistence.

---

#### Domain Service (Service de domaine)

Un Domain Service encapsule une opération métier significative qui ne trouve
naturellement sa place dans aucune entité ou Value Object — généralement parce
qu'elle implique plusieurs agrégats ou parce qu'elle est sans état.

**A ne pas confondre avec :**
- le *Application Service* (orchestration des cas d'usage, couche applicative) ;
- le *Infrastructure Service* (e-mail, SMS, connexion BDD — couche infrastructure).

```php
<?php

declare(strict_types=1);

/**
 * Service de domaine : transfert de fonds entre deux comptes.
 * L'opération implique deux agrégats (Account) — elle ne peut pas
 * appartenir à l'un ou à l'autre sans arbitraire.
 */
final class FundTransferService
{
    public function transfer(
        Account $source,
        Account $destination,
        Money   $amount,
    ): void {
        if (!$source->currency()->equals($destination->currency())) {
            throw new \DomainException(
                "Le transfert entre devises différentes n'est pas supporté."
            );
        }
        if (!$source->hasSufficientFunds($amount)) {
            throw new \DomainException(
                "Fonds insuffisants sur le compte source."
            );
        }

        $source->debit($amount);
        $destination->credit($amount);
    }
}
```

---

### 3.3 Bounded Context — frontières du modèle

#### Définition

Un Bounded Context (contexte borné) est la frontière dans laquelle un modèle
de domaine particulier s'applique de façon cohérente. A l'intérieur d'un Bounded
Context, chaque terme a une définition précise et unique. Un même mot peut avoir
une signification différente dans un autre Bounded Context.

**Exemple :** le mot « client » dans un système e-commerce :

| Bounded Context | Ce que signifie « client » |
|---|---|
| Ventes | Personne qui passe une commande ; attributs : nom, adresse livraison, historique achats |
| Service client | Interlocuteur avec tickets de support ; attributs : numéro de contrat, niveau SLA |
| Comptabilité | Débiteur dans le grand livre ; attributs : SIRET, RIB, conditions de paiement |
| Marketing | Segment d'audience ; attributs : comportement d'achat, score NPS |

Si un seul modèle `Customer` essaie de satisfaire ces quatre contextes, il
deviendra une « God Class » avec des dizaines d'attributs optionnels, difficile
à comprendre et à faire évoluer.

#### Context Map — cartographier les relations entre contextes

La Context Map est la représentation des relations entre les Bounded Contexts
d'un système. Evans définit plusieurs types de relations :

**Shared Kernel :** deux équipes partagent un sous-modèle commun. Toute
modification requiert un accord des deux équipes. Couplage fort.

**Customer-Supplier :** un contexte aval (Customer) dépend d'un contexte amont
(Supplier). L'amont définit le contrat ; l'aval s'y adapte.

**Conformist :** l'aval adopte le modèle de l'amont sans négociation (cas d'un
système tiers imposé : une API bancaire, un ERP).

**Anti-Corruption Layer (ACL) :** l'aval traduit le modèle de l'amont en son
propre langage grâce à une couche de traduction. C'est le pattern le plus protecteur
de l'intégrité du modèle.

```python
# Anti-Corruption Layer : le contexte Commandes traduit le modèle du contexte
# Inventaire (système legacy) sans laisser ses structures contaminer le domaine

class InventoryLegacyClient:
    """Appelle le système d'inventaire legacy (API SOAP années 2000)."""

    def get_stock_level(self, sku: str) -> dict:
        # Retourne {'SKU': 'PRD-001', 'QTE_DISPO': 42, 'STATUT_ART': 'A'}
        ...


class InventoryAntiCorruptionLayer:
    """
    Traduit le modèle legacy en concepts du domaine Commandes.
    Le domaine Commandes ne connaît jamais InventoryLegacyClient directement.
    """

    def __init__(self, legacy_client: InventoryLegacyClient):
        self._client = legacy_client

    def is_product_available(self, product_id: ProductId, quantity: int) -> bool:
        raw = self._client.get_stock_level(product_id.value)
        # Traduction : STATUT_ART 'A' = actif, QTE_DISPO = stock disponible
        is_active      = raw.get("STATUT_ART") == "A"
        available_qty  = int(raw.get("QTE_DISPO", 0))
        return is_active and available_qty >= quantity
```

**Open Host Service / Published Language :** un contexte expose une API publique
documentée que plusieurs autres consomment. Les API REST avec un contrat OpenAPI
en sont l'illustration moderne.

---

### 3.4 DDD tactique vs stratégique — quand l'appliquer

Le DDD se décompose en deux niveaux :

**DDD stratégique** : Ubiquitous Language, Bounded Contexts, Context Map,
Subdomains. S'applique à *tout* projet non trivial, quelle que soit la taille.
Le coût est faible (discussions, documentation) ; le bénéfice est élevé (alignement
équipe–métier, frontières claires).

**DDD tactique** : Entities, Value Objects, Aggregates, Repositories, Domain
Services, Domain Events. Ne se justifie que dans les sous-domaines au cœur du
business (*core domain*) où la complexité métier est élevée.

Le DDD distingue trois types de sous-domaines :

| Type | Description | Approche recommandée |
|---|---|---|
| **Core Domain** | Avantage concurrentiel différenciant | DDD tactique complet |
| **Supporting Domain** | Soutient le core, pas différenciant | DDD tactique partiel ou CRUD enrichi |
| **Generic Domain** | Commodité standard (authentification, facturation simple) | Solution standard du marché ou CRUD |

**Avertissement :** appliquer le DDD tactique complet à un simple CRUD
(formulaire → base de données → liste) est de la sur-ingénierie. Le DDD est
un outil pour gérer la complexité métier — s'il n'y a pas de complexité métier,
le simple CRUD est la bonne réponse.

---

### 3.5 Lien avec les principes SOLID

Les patterns DDD et les principes SOLID ne sont pas indépendants. Voici les
correspondances les plus directes :

| Pattern DDD | Principe SOLID | Explication |
|---|---|---|
| Aggregate | SRP | L'Aggregate définit une frontière de cohérence : une seule raison de changer (les règles métier de ce groupe d'entités). |
| Repository | DIP | L'interface est dans le domaine ; l'implémentation dans l'infrastructure. Le domaine ne dépend pas de la BDD. |
| Value Object | OCP + SRP | Immuable et auto-validant : fermé à la modification (une fois créé, ses données ne changent pas), ouvert à l'extension (nouvelles opérations via de nouvelles méthodes). |
| Domain Service | SRP | Isole une opération qui implique plusieurs agrégats, évitant de surcharger l'un d'eux. |
| Anti-Corruption Layer | DIP + LSP | Traduit le modèle externe en abstractions internes. Le domaine dépend de l'ACL (abstraction), pas du système tiers (concrétion). |
| Bounded Context | ISP | Chaque contexte expose uniquement le modèle pertinent pour ses consommateurs — pas un modèle universel tentant de servir tout le monde. |

---

## Session 4 — Atelier : Event Storming & modélisation du domaine

### 4.1 Qu'est-ce que l'Event Storming ?

L'Event Storming est une technique d'atelier collaboratif inventée par Alberto
Brandolini en 2013. Son objectif : explorer rapidement un domaine métier complexe
en rassemblant dans la même pièce des développeurs et des experts métier, armés
de post-its et d'un grand mur.

Le principe fondateur de Brandolini :

> « La quantité de connaissances du domaine contenue dans la tête des experts
> métier est toujours supérieure à ce que peut absorber n'importe quel développeur
> seul. L'Event Storming permet de les externaliser collectivement. »

L'Event Storming est délibérément *non structuré* dans ses premières phases :
on libère la parole avant de structurer les résultats.

---

### 4.2 Les éléments de l'Event Storming

Chaque type d'information est représenté par une couleur de post-it différente.
La convention la plus répandue (celle de Brandolini) est la suivante :

| Couleur | Type | Description |
|---|---|---|
| Orange | **Domain Event** | Quelque chose qui s'est passé dans le domaine, formé au passé : *Commande confirmée*, *Paiement échoué*, *Article expédié* |
| Bleu | **Command** | Ce qui déclenche un Domain Event, formulé à l'impératif : *Confirmer la commande*, *Effectuer le paiement* |
| Jaune clair | **Actor** | L'humain ou le système qui émet la commande : *Client*, *Système de paiement* |
| Jaune | **Aggregate** | L'entité métier sur laquelle s'exerce la commande et qui produit l'événement |
| Violet | **Policy / Reaction** | Règle métier déclenchée par un événement : *Quand la commande est confirmée, envoyer un e-mail de confirmation* |
| Rouge | **Hot spot** | Question, ambiguïté ou problème à résoudre plus tard |
| Vert | **Read Model** | Information que l'acteur consulte pour prendre sa décision |
| Rose | **External System** | Système externe impliqué dans le flux |

---

### 4.3 Déroulement d'un atelier Event Storming

#### Phase 1 — Divergence (20 min) : les événements en vrac

Chaque participant écrit des Domain Events sur des post-its orange, sans
contrainte d'ordre ni de complétude. La règle : formuler au passé composé.

Exemples pour un système de location de véhicules :
- *Réservation créée*
- *Véhicule attribué*
- *Contrat signé*
- *Clés remises*
- *Véhicule rendu*
- *Dommage constaté*
- *Caution débitée*
- *Facture générée*
- *Paiement reçu*
- *Avis client publié*

On pose tous les post-its sur le mur sans discussion, dans le désordre.

#### Phase 2 — Convergence (20 min) : tri et ligne de temps

L'animateur invite les participants à déplacer les post-its pour former une
ligne de temps de gauche à droite. Les doublons sont regroupés ; les
contradictions donnent lieu à un post-it rouge (*hot spot*).

#### Phase 3 — Enrichissement (30 min) : commandes, acteurs, agrégats

Pour chaque Domain Event significatif :
- Qui ou quoi l'a déclenché ? → Command (bleu) + Actor (jaune clair)
- Sur quelle entité métier ? → Aggregate (jaune)
- Quelle réaction en découle ? → Policy (violet)

```
[Client] → [Créer réservation] → (Réservation) → *Réservation créée*
                                                         ↓
                                    [Politique] Quand réservation créée,
                                    vérifier disponibilité du véhicule
                                                         ↓
                              [Système disponibilité] → *Véhicule attribué*
                                                    ou *Aucun véhicule disponible*
```

#### Phase 4 — Frontières (20 min) : Bounded Contexts

On trace des lignes autour des groupes d'événements qui semblent appartenir au
même langage et aux mêmes règles. Chaque groupe est un candidat de Bounded Context.

Exemple pour la location de véhicules :
- **Réservation** : *Réservation créée*, *Véhicule attribué*, *Réservation annulée*
- **Contrat** : *Contrat signé*, *Clés remises*, *Conditions acceptées*
- **Retour & sinistres** : *Véhicule rendu*, *Dommage constaté*, *Caution débitée*
- **Facturation** : *Facture générée*, *Paiement reçu*, *Remboursement effectué*

---

### 4.4 Du mur de post-its au code squelette

L'Event Storming produit un modèle collaboratif. La traduction en code suit un
mapping direct entre les éléments visuels et les concepts DDD.

#### Mapping Event Storming → Code

| Post-it Event Storming | Concept DDD | Implémentation |
|---|---|---|
| Domain Event (orange) | `DomainEvent` | Classe immuable avec timestamp |
| Aggregate (jaune) | `Aggregate Root` | Classe avec méthodes métier |
| Command (bleu) | Méthode de l'agrégat ou Application Service | Méthode ou classe de commande |
| Policy (violet) | `DomainEventHandler` / Application Service | Abonné à un bus d'événements |
| Read Model (vert) | `Query` / `DTO` | Projection de lecture |
| Bounded Context | Module / Namespace / Package | Organisation de répertoires |

#### Exemple complet : du post-it au code

Nous allons modéliser le flux *Réservation* du système de location.

**Post-its issus de l'atelier :**

```
[Client] → [Créer réservation] → (Réservation) → *Réservation créée*
[Client] → [Annuler réservation] → (Réservation) → *Réservation annulée*
[Politique] Quand *Réservation créée* → [Système inventaire] → *Véhicule attribué*
[Politique] Quand *Réservation créée* → [Service e-mail] → *E-mail de confirmation envoyé*
```

**Code squelette résultant :**

```php
<?php

declare(strict_types=1);

// ── Domain Events ─────────────────────────────────────────────────────────────

interface DomainEvent
{
    public function occurredAt(): \DateTimeImmutable;
}

final class ReservationCreated implements DomainEvent
{
    public function __construct(
        public readonly ReservationId $reservationId,
        public readonly CustomerId    $customerId,
        public readonly VehicleTypeId $requestedType,
        public readonly DateRange     $period,
        private readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}

    public function occurredAt(): \DateTimeImmutable { return $this->occurredAt; }
}

final class ReservationCancelled implements DomainEvent
{
    public function __construct(
        public readonly ReservationId $reservationId,
        public readonly string        $reason,
        private readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}

    public function occurredAt(): \DateTimeImmutable { return $this->occurredAt; }
}

final class VehicleAllocated implements DomainEvent
{
    public function __construct(
        public readonly ReservationId $reservationId,
        public readonly VehicleId     $vehicleId,
        private readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}

    public function occurredAt(): \DateTimeImmutable { return $this->occurredAt; }
}


// ── Value Objects ─────────────────────────────────────────────────────────────

final class ReservationId
{
    public function __construct(public readonly string $value)
    {
        if (empty($value)) throw new \DomainException("ReservationId ne peut pas être vide.");
    }

    public static function generate(): self
    {
        return new self(sprintf('RES-%s', strtoupper(substr(uniqid('', true), -8))));
    }
}

final class DateRange
{
    public function __construct(
        public readonly \DateTimeImmutable $start,
        public readonly \DateTimeImmutable $end,
    ) {
        if ($end <= $start) {
            throw new \DomainException("La date de fin doit être postérieure à la date de début.");
        }
    }

    public function durationDays(): int
    {
        return (int) $this->start->diff($this->end)->days;
    }

    public function overlaps(self $other): bool
    {
        return $this->start < $other->end && $this->end > $other->start;
    }
}


// ── Aggregate Root ────────────────────────────────────────────────────────────

enum ReservationStatus { case PENDING; case CONFIRMED; case CANCELLED; }

class Reservation       // Aggregate Root issu du post-it jaune
{
    private ReservationStatus $status;

    /** @var DomainEvent[] */
    private array $domainEvents = [];

    private ?VehicleId $allocatedVehicle = null;

    public function __construct(
        private readonly ReservationId $id,
        private readonly CustomerId    $customerId,
        private readonly VehicleTypeId $requestedType,
        private readonly DateRange     $period,
    ) {
        $this->status = ReservationStatus::PENDING;
        // La création produit un événement — cf. post-it *Réservation créée*
        $this->recordEvent(new ReservationCreated(
            $this->id, $this->customerId, $this->requestedType, $this->period
        ));
    }

    // ── Commandes (post-its bleus) ────────────────────────────────────────────

    public function allocateVehicle(VehicleId $vehicleId): void
    {
        if ($this->status !== ReservationStatus::PENDING) {
            throw new \DomainException("Impossible d'attribuer un véhicule à cette réservation.");
        }
        $this->allocatedVehicle = $vehicleId;
        $this->status           = ReservationStatus::CONFIRMED;
        $this->recordEvent(new VehicleAllocated($this->id, $vehicleId));
    }

    public function cancel(string $reason): void
    {
        if ($this->status === ReservationStatus::CANCELLED) {
            throw new \DomainException("La réservation est déjà annulée.");
        }
        $this->status = ReservationStatus::CANCELLED;
        $this->recordEvent(new ReservationCancelled($this->id, $reason));
    }

    // ── Accesseurs ────────────────────────────────────────────────────────────

    public function id(): ReservationId           { return $this->id; }
    public function customerId(): CustomerId      { return $this->customerId; }
    public function period(): DateRange           { return $this->period; }
    public function status(): ReservationStatus   { return $this->status; }

    /** @return DomainEvent[] */
    public function pullDomainEvents(): array
    {
        $events             = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    private function recordEvent(DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }
}


// ── Repository Interface (dans la couche domaine) ─────────────────────────────

interface ReservationRepository
{
    public function findById(ReservationId $id): ?Reservation;

    /** @return Reservation[] */
    public function findActiveByCustomer(CustomerId $customerId): array;

    /** @return Reservation[] */
    public function findConflicting(VehicleTypeId $type, DateRange $period): array;

    public function save(Reservation $reservation): void;
}


// ── Application Service (orchestration du cas d'usage) ────────────────────────

final class CreateReservationCommand
{
    public function __construct(
        public readonly string $customerId,
        public readonly string $vehicleTypeId,
        public readonly string $startDate,
        public readonly string $endDate,
    ) {}
}

final class CreateReservationHandler
{
    public function __construct(
        private readonly ReservationRepository $reservations,
        private readonly DomainEventBus        $eventBus,
    ) {}

    public function handle(CreateReservationCommand $command): ReservationId
    {
        $period = new DateRange(
            new \DateTimeImmutable($command->startDate),
            new \DateTimeImmutable($command->endDate),
        );

        // Règle métier : pas de réservation de moins d'un jour
        if ($period->durationDays() < 1) {
            throw new \DomainException("La durée minimale de réservation est d'un jour.");
        }

        $reservation = new Reservation(
            ReservationId::generate(),
            new CustomerId($command->customerId),
            new VehicleTypeId($command->vehicleTypeId),
            $period,
        );

        $this->reservations->save($reservation);

        // Publication des Domain Events (→ déclenche les Policies/Reactions)
        foreach ($reservation->pullDomainEvents() as $event) {
            $this->eventBus->publish($event);
        }

        return $reservation->id();
    }
}


// ── Policy / Reaction (post-its violets) ──────────────────────────────────────

/**
 * Politique : quand une réservation est créée, attribuer un véhicule.
 * Découplée de CreateReservationHandler — elle réagit à l'événement.
 */
final class AllocateVehicleOnReservationCreated
{
    public function __construct(
        private readonly ReservationRepository  $reservations,
        private readonly VehicleInventoryPort   $inventory,
        private readonly DomainEventBus         $eventBus,
    ) {}

    public function __invoke(ReservationCreated $event): void
    {
        $vehicle = $this->inventory->findAvailable(
            $event->requestedType,
            $event->period,
        );

        if ($vehicle === null) {
            // Pas de véhicule disponible : on pourrait publier un autre événement
            // (NoVehicleAvailable) qui déclencherait une notification client.
            return;
        }

        $reservation = $this->reservations->findById($event->reservationId);
        $reservation->allocateVehicle($vehicle->id());
        $this->reservations->save($reservation);

        foreach ($reservation->pullDomainEvents() as $domainEvent) {
            $this->eventBus->publish($domainEvent);
        }
    }
}
```

---

### 4.5 CRUD classique vs approche orientée domaine

Il est instructif de comparer les deux approches sur le même cas pour en percevoir
les différences concrètes.

**Cas d'usage :** confirmer une commande en vérifiant que le stock est suffisant,
en débitant ce stock et en envoyant un e-mail de confirmation.

#### Approche CRUD

```python
# Contrôleur CRUD typique
def confirm_order(order_id: int, db: Session) -> dict:
    order = db.query(OrderModel).get(order_id)
    if not order:
        return {"error": "Order not found"}, 404

    items = db.query(OrderItemModel).filter_by(order_id=order_id).all()
    for item in items:
        product = db.query(ProductModel).get(item.product_id)
        if product.stock < item.quantity:
            return {"error": f"Insufficient stock for {product.name}"}, 400
        product.stock -= item.quantity

    order.status = "confirmed"
    order.confirmed_at = datetime.utcnow()
    db.commit()

    send_email(order.customer_email, "Order confirmed", f"Order {order_id} confirmed.")

    return {"status": "confirmed"}
```

**Problèmes :**
- La logique métier (règles de confirmation, vérification du stock) est dans le
  contrôleur — impossible à tester sans HTTP et base de données.
- Pas d'Ubiquitous Language : `status = "confirmed"` plutôt que `order.confirm()`.
- Les effets de bord (email, décrémentation du stock) sont mélangés à la transaction.
- Si la règle de confirmation change, on modifie le contrôleur.

#### Approche orientée domaine

```python
# Application Service — orchestration
class ConfirmOrderHandler:
    def __init__(
        self,
        orders:    OrderRepository,
        inventory: InventoryService,
        notifier:  OrderNotifier,
        events:    DomainEventBus,
    ):
        self._orders    = orders
        self._inventory = inventory
        self._notifier  = notifier
        self._events    = events

    def handle(self, command: ConfirmOrderCommand) -> None:
        order = self._orders.find_by_id(OrderId(command.order_id))
        if order is None:
            raise OrderNotFound(command.order_id)

        # Vérification du stock (Domain Service)
        self._inventory.ensure_stock_available(order)

        # Comportement métier encapsulé dans l'agrégat
        order.confirm()

        # Persistence et publication des événements
        self._orders.save(order)
        for event in order.pull_domain_events():
            self._events.publish(event)

# Agrégat — règles métier
class Order:
    def confirm(self) -> None:
        if self._status != OrderStatus.PENDING:
            raise DomainException("Seules les commandes en attente peuvent être confirmées.")
        if not self._lines:
            raise DomainException("Une commande vide ne peut pas être confirmée.")
        self._status = OrderStatus.CONFIRMED
        self._confirmed_at = datetime.utcnow()
        self._record_event(OrderConfirmed(self._id, self._customer_id, self._confirmed_at))

# Policy — réaction à l'événement
class SendConfirmationEmailOnOrderConfirmed:
    def __call__(self, event: OrderConfirmed) -> None:
        self._mailer.send_order_confirmation(event.customer_id, event.order_id)
```

**Bénéfices :**
- `order.confirm()` est testable en isolation, sans BDD ni HTTP.
- La règle métier vit dans l'agrégat — le contrôleur/handler est mince.
- L'email est découplé de la transaction via un événement.
- Changer la règle de confirmation ne touche que la classe `Order`.

---

### 4.6 Atelier guidé — modélisation d'un système de bibliothèque

#### Contexte

Vous êtes chargés de modéliser le système informatique d'une médiathèque publique.
L'atelier se déroule en deux temps : d'abord l'Event Storming (sur papier ou tableau
blanc), puis la traduction en code squelette.

#### Description du domaine

La médiathèque propose des livres, des DVDs et des magazines. Les adhérents peuvent
emprunter des documents pour une durée variable selon le type (livre : 3 semaines,
DVD : 1 semaine, magazine : 1 semaine). Un adhérent peut avoir au maximum 5
documents empruntés simultanément.

Si un document est rendu en retard, une pénalité est calculée (0,10 € par jour de
retard). Si les pénalités dépassent 5 €, l'adhérent est suspendu jusqu'au règlement.

Certains documents sont en exemplaire unique et peuvent être réservés s'ils ne sont
pas disponibles. Quand un document réservé devient disponible (rendu par son
emprunteur), l'adhérent en attente est notifié par email.

---

#### Partie 1 — Event Storming (40 min, en groupes de 4)

**Etape 1 (10 min) :** Chaque membre du groupe écrit individuellement tous les
Domain Events qu'il identifie dans ce domaine. Minimum 10 événements par groupe.

**Etape 2 (10 min) :** Regrouper les post-its sur une ligne de temps. Identifier
les doublons, les ambiguïtés (*hot spots* en rouge), les événements manquants.

**Etape 3 (20 min) :** Pour chaque événement significatif, ajouter :
- la Command qui le déclenche (bleu) ;
- l'Actor (jaune clair) ;
- l'Aggregate concerné (jaune) ;
- les éventuelles Policies (violet).

Délimiter les Bounded Contexts.

---

#### Partie 2 — Code squelette (30 min, en groupes de 4)

A partir de votre Event Storm, produire le code squelette (en PHP ou Python)
incluant pour chacun des Bounded Contexts identifiés :

1. Les Value Objects du contexte.
2. L'Aggregate Root avec ses méthodes métier (corps vides acceptés).
3. Les Domain Events produits par l'agrégat.
4. L'interface du Repository.
5. Un Application Service pour le cas d'usage principal du contexte.


## Résumé

| Concept | Ce qu'il apporte | Relation avec le matin |
|---|---|---|
| Ubiquitous Language | Pont entre code et métier | Prolonge le nommage expressif (Clean Code) |
| Value Object | Encapsulation des règles de validation et des opérations | Elimine la Primitive Obsession |
| Entity | Identité stable, mutations contrôlées | Applique le SRP et POLA |
| Aggregate | Frontière de cohérence transactionnelle | Applique SRP, élimine les incohérences |
| Repository | Abstraction de la persistence | Application directe du DIP |
| Domain Service | Opérations impliquant plusieurs agrégats | Application du SRP |
| Bounded Context | Frontière du modèle et du langage | Application de l'ISP à l'échelle du système |
| Anti-Corruption Layer | Protection de l'intégrité face aux systèmes externes | Application du DIP + LSP |
| Event Storming | Exploration collaborative du domaine | Produit l'Ubiquitous Language de façon organique |
| Domain Events | Communication découplée entre contextes | Prépare CQRS et Event Sourcing (jour 2) |

Le lendemain matin abordera la programmation fonctionnelle comme complément
au paradigme objet, avant de conclure l'après-midi sur les architectures
hexagonale et Clean Architecture — qui formalisent ce que le DDD nomme
la séparation entre domaine et infrastructure.

---

## Bibliographie

### Ouvrages fondateurs du DDD

**Evans, E.** (2003). *Domain-Driven Design: Tackling Complexity in the Heart of Software*.
Addison-Wesley.  
Le livre fondateur, surnommé le « Big Blue Book ». Il introduit l'intégralité du vocabulaire
présenté cet après-midi : Ubiquitous Language, Entity, Value Object, Aggregate, Repository,
Domain Service, Bounded Context, Context Map, Anti-Corruption Layer. Dense et parfois aride,
il se lit mieux en deux passages : une lecture rapide pour saisir la vue d'ensemble, puis une
relecture approfondie chapitre par chapitre. Les chapitres 1 à 3 (langage et modèle), 5 à 7
(blocs de construction tactiques) et 14 à 16 (conception stratégique) sont les plus directement
utiles pour cette session.

**Vernon, V.** (2013). *Implementing Domain-Driven Design*. Addison-Wesley.  
Le « Big Red Book », compagnon pratique indispensable d'Evans. Là où Evans pose les concepts,
Vernon les illustre par du code complet. Le chapitre 5 (Entities), le chapitre 6 (Value Objects)
et le chapitre 10 (Aggregates, avec les quatre règles de conception) couvrent précisément ce qui
a été présenté en session 3. Le chapitre 3 (Bounded Contexts et Context Maps) approfondit la
cartographie stratégique.

**Vernon, V.** (2016). *Domain-Driven Design Distilled*. Addison-Wesley.  
Version condensée (250 pages) conçue pour être lue en une journée. Idéale comme première
approche avant d'aborder les deux ouvrages précédents, ou comme révision rapide. Couvre
l'Event Storming, les Bounded Contexts et les patterns tactiques essentiels sans la densité
théorique du Big Blue Book.

**Millett, S. & Tune, N.** (2015). *Patterns, Principles, and Practices of Domain-Driven Design*.
Wrox Press.  
Alternative pédagogique aux livres d'Evans et Vernon, avec un accent mis sur la transition
depuis une architecture en couches classique vers le DDD. Particulièrement utile pour des
étudiants ayant déjà une expérience professionnelle : les exemples partent de code existant
et montrent comment le DDD s'y introduit progressivement.

### Event Storming

**Brandolini, A.** (2021). *Introducing Event Storming*. Leanpub.  
Le livre de référence sur l'Event Storming, écrit par son inventeur. Disponible en version
numérique sur Leanpub, régulièrement mis à jour. Couvre les trois niveaux d'un atelier
(Big Picture, Process Level, Design Level) et les conditions de succès ou d'échec d'un
atelier réel. La lecture des cinquante premières pages suffit pour animer la session 4.

**Brandolini, A.** (2013). Introducing Event Storming. *ziobrando.blogspot.com*.  
L'article de blog original dans lequel Brandolini présente l'Event Storming pour la première
fois. Court et accessible, il expose l'intuition fondatrice : réunir développeurs et experts
métier autour de post-its orange pour explorer un domaine sans biais technologique.

### Articles et ressources en ligne

**Fowler, M.** (2003). *Bounded Context*. MartinFowler.com.  
Entrée du bliki de Martin Fowler consacrée au Bounded Context. Synthétique, elle offre
une définition alternative à celle d'Evans et plusieurs exemples concrets.
https://martinfowler.com/bliki/BoundedContext.html

**Fowler, M.** (2002). *Domain Model*. MartinFowler.com — extrait de *Patterns of Enterprise
Application Architecture*. Addison-Wesley.  
Décrit le pattern Domain Model en opposition au Transaction Script et au Table Module.
Utile pour expliquer pourquoi une approche orientée objet riche est préférable au CRUD
dans les contextes à forte complexité métier.

**Fowler, M., Rice, D., Foemmel, M., Hieatt, E., Mee, R. & Stafford, R.** (2002).
*Patterns of Enterprise Application Architecture*. Addison-Wesley.  
Contient les descriptions originales de plusieurs patterns discutés en session 3 : Repository,
Unit of Work, Identity Map. Ouvrage de référence sur les patterns d'architecture d'entreprise,
antérieur au DDD mais complémentaire.

### Pour aller plus loin

**Young, G.** (2010). CQRS Documents. Disponible sur https://cqrs.files.wordpress.com  
Greg Young est l'un des principaux contributeurs à la popularisation du DDD dans la
communauté .NET/Java. Ce document de synthèse introduit CQRS et Event Sourcing comme
prolongements naturels du DDD, thèmes abordés le lendemain après-midi.

**Richardson, C.** (2018). *Microservices Patterns*. Manning Publications.  
Le chapitre 2 de cet ouvrage applique le DDD — et notamment les Bounded Contexts — à
la décomposition en microservices. Utile pour les étudiants qui travailleront dans des
architectures distribuées et souhaitent ancrer le DDD dans un contexte opérationnel concret.
