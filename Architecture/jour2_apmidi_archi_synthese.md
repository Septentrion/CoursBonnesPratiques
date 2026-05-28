# Jour 2 — Après-midi : Architecture évolutive, pragmatisme & synthèse

**Durée totale :** 3 h 30 (cours magistral, débat structuré, synthèse)  
**Public :** Etudiants en informatique — dernière session de la formation  
**Langages illustrés :** PHP 8.x et Python 3.10+

---

## Avant-propos : assembler les pièces

Les deux dernières journées ont progressé du micro au macro :

```
Nommage, fonctions         → Clean Code (jour 1 matin)
Refactoring, Mikado        → Maîtrise du code legacy (jour 1 matin)
DDD tactique & stratégique → Modélisation du domaine (jour 1 après-midi)
Event Storming             → Exploration collaborative (jour 1 après-midi)
Programmation fonctionnelle → Transformation de données (jour 2 matin)
TDD, tests avancés         → Filet de sécurité (jour 2 matin)
```

Cette dernière demi-journée monte au niveau le plus large : comment organiser
l'ensemble d'une application pour que toutes les pratiques précédentes s'y
inscrivent naturellement ? Les architectures hexagonale et Clean Architecture
répondent à cette question. CQRS et Event Sourcing la prolongent pour les
systèmes à haute complexité. Le débat structuré et la synthèse terminent la
formation sur la question la plus importante : *comment choisir ce qu'on applique,
et quand ne pas l'appliquer* ?

---

## Session 7 — Architecture hexagonale & évolutive (13 h 30 – 15 h)

### 7.1 Architecture hexagonale — Ports & Adapters

#### Origine et intention

L'architecture hexagonale a été formulée par Alistair Cockburn en 2005 dans un
article fondateur : *Hexagonal Architecture — Allow an application to equally
be driven by users, programs, automated tests or batch scripts, and to be
developed and tested in isolation from its eventual run-time devices and
databases.*

L'intention est précise : une application doit pouvoir être testée **complètement
en isolation** de son infrastructure (BDD, HTTP, système de fichiers, services
externes) sans que cela nécessite de modifier son code métier.

L'hexagone lui-même n'a rien de mathématiquement significatif — il illustre
simplement qu'une application a plusieurs faces par lesquelles elle interagit
avec le monde extérieur, et qu'aucune face n'est plus importante que les autres.

#### Structure

```
                    ┌─────────────────────────────────────────┐
  Tests unitaires   │                                         │  Interface web
  Scripts CLI  ────►│         APPLICATION (Hexagone)          │◄──── REST API
  Tests E2E         │                                         │      GraphQL
                    │   Logique métier pure, Aggregates,      │
  Queue de msgs ───►│   Domain Services, Application Services │◄──── Message bus
  Tâches planifiées │                                         │
                    └──────────────┬──────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────┐
                    │          PORTS (interfaces)              │
                    │  OrderRepositoryPort  NotifierPort       │
                    │  PaymentGatewayPort   SearchPort         │
                    └──────────────┬──────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────┐
                    │         ADAPTERS (implémentations)       │
                    │  SqlOrderRepository  SmtpNotifier        │
                    │  StripeGateway       ElasticSearchAdapter│
                    └─────────────────────────────────────────┘
```

**Ports :** interfaces définies *par* l'application, exprimant ce dont elle a
besoin. Elles font partie de la couche domaine ou applicative.

**Adapters :** implémentations des ports qui traduisent entre le monde extérieur
et le langage de l'application. Ils font partie de la couche infrastructure.

On distingue deux types de ports/adapters :

- **Ports primaires** (*driving side*, côté gauche) : l'interface par laquelle
  des acteurs externes *pilotent* l'application — contrôleurs HTTP, CLI, tests.
- **Ports secondaires** (*driven side*, côté droit) : l'interface par laquelle
  l'application *pilote* des systèmes externes — BDD, API tiers, services e-mail.

#### Implémentation

```python
# ── Structure de répertoires ──────────────────────────────────────────────────
#
# src/
# ├── domain/                      ← Hexagone (pure, sans dépendances)
# │   ├── model/
# │   │   ├── order.py             ← Aggregate
# │   │   └── order_line.py
# │   ├── ports/
# │   │   ├── order_repository.py  ← Port secondaire (interface)
# │   │   └── payment_gateway.py   ← Port secondaire (interface)
# │   └── services/
# │       └── order_service.py     ← Domain Service
# │
# ├── application/                 ← Cas d'usage (orchestration)
# │   ├── commands/
# │   │   └── place_order.py       ← Command + Handler
# │   └── queries/
# │       └── get_order.py         ← Query + Handler
# │
# └── infrastructure/              ← Adapters
#     ├── persistence/
#     │   └── sql_order_repository.py   ← Adapter secondaire
#     ├── http/
#     │   └── order_controller.py       ← Adapter primaire (Flask/FastAPI)
#     └── payment/
#         └── stripe_gateway.py         ← Adapter secondaire


# ── Port secondaire — défini dans la couche domaine ──────────────────────────

from abc import ABC, abstractmethod
from domain.model.order import Order, OrderId


class OrderRepositoryPort(ABC):
    """
    Port secondaire : contrat que tout système de persistence d'Order doit respecter.
    Défini par le domaine, implémenté par l'infrastructure.
    """

    @abstractmethod
    def find_by_id(self, order_id: OrderId) -> Order | None: ...

    @abstractmethod
    def save(self, order: Order) -> None: ...


class PaymentGatewayPort(ABC):
    """Port secondaire : abstraction du système de paiement."""

    @abstractmethod
    def charge(
        self,
        amount_cents: int,
        currency: str,
        card_token: str,
    ) -> "PaymentResult": ...


# ── Application Service — dans la couche application ─────────────────────────

from dataclasses import dataclass
from domain.ports.order_repository import OrderRepositoryPort
from domain.ports.payment_gateway  import PaymentGatewayPort


@dataclass(frozen=True)
class PlaceOrderCommand:
    customer_id: str
    product_id:  str
    quantity:    int
    card_token:  str


class PlaceOrderHandler:
    """
    Adapter primaire interne : orchestre un cas d'usage.
    Dépend uniquement des Ports (abstractions), jamais des Adapters.
    """

    def __init__(
        self,
        orders:  OrderRepositoryPort,
        gateway: PaymentGatewayPort,
    ):
        self._orders  = orders
        self._gateway = gateway

    def handle(self, command: PlaceOrderCommand) -> OrderId:
        order = Order.create(
            customer_id=CustomerId(command.customer_id),
            product_id=ProductId(command.product_id),
            quantity=command.quantity,
        )
        payment = self._gateway.charge(
            order.total_cents(), "EUR", command.card_token
        )
        order.confirm(payment.transaction_id)
        self._orders.save(order)
        return order.id


# ── Adapter primaire HTTP — dans la couche infrastructure ─────────────────────

# Flask example
from flask import Flask, request, jsonify
from application.commands.place_order import PlaceOrderCommand, PlaceOrderHandler

app = Flask(__name__)

@app.route("/orders", methods=["POST"])
def create_order():
    body = request.json
    handler = PlaceOrderHandler(
        orders=app.config["ORDER_REPO"],
        gateway=app.config["PAYMENT_GATEWAY"],
    )
    command  = PlaceOrderCommand(
        customer_id=body["customer_id"],
        product_id=body["product_id"],
        quantity=body["quantity"],
        card_token=body["card_token"],
    )
    order_id = handler.handle(command)
    return jsonify({"order_id": str(order_id.value)}), 201


# ── Adapter secondaire SQL — dans la couche infrastructure ────────────────────

import sqlite3
from domain.ports.order_repository import OrderRepositoryPort
from domain.model.order import Order, OrderId


class SqliteOrderRepository(OrderRepositoryPort):
    """Implémentation du Port secondaire OrderRepositoryPort via SQLite."""

    def __init__(self, connection: sqlite3.Connection):
        self._conn = connection

    def find_by_id(self, order_id: OrderId) -> Order | None:
        row = self._conn.execute(
            "SELECT * FROM orders WHERE id = ?", (order_id.value,)
        ).fetchone()
        return self._hydrate(row) if row else None

    def save(self, order: Order) -> None:
        self._conn.execute(
            "INSERT OR REPLACE INTO orders (id, customer_id, status, total_cents) "
            "VALUES (?, ?, ?, ?)",
            (order.id.value, str(order.customer_id),
             order.status.name, order.total_cents()),
        )
        self._conn.commit()

    def _hydrate(self, row) -> Order:
        ...  # reconstruction de l'Aggregate depuis la BDD


# ── Test sans infrastructure : on branche des Fakes sur les Ports ─────────────

class FakeOrderRepository(OrderRepositoryPort):
    def __init__(self): self._store: dict = {}
    def find_by_id(self, oid): return self._store.get(oid.value)
    def save(self, order): self._store[order.id.value] = order

class FakePaymentGateway(PaymentGatewayPort):
    def charge(self, amount_cents, currency, card_token):
        return PaymentResult(success=True, transaction_id="TX-FAKE-001")

def test_place_order_succeeds():
    repo    = FakeOrderRepository()
    gateway = FakePaymentGateway()
    handler = PlaceOrderHandler(repo, gateway)

    command  = PlaceOrderCommand("C1", "P1", 2, "tok_test")
    order_id = handler.handle(command)

    assert repo.find_by_id(order_id) is not None
```

---

### 7.2 Clean Architecture — la règle de dépendance

#### Origine

Robert C. Martin (Uncle Bob) a formalisé la Clean Architecture en 2012 comme
une synthèse de plusieurs architectures en couches : Hexagonale (Cockburn),
Onion (Jeffrey Palermo), BCE (*Boundary-Control-Entity*, Jacobson). Toutes
partagent le même objectif et la même règle fondamentale.

#### La règle de dépendance

> « Les dépendances du code source ne pointent que vers l'intérieur. »

```
┌─────────────────────────────────────────────────┐
│  Frameworks & Drivers                           │  Couche externe
│  (HTTP, BDD, UI, tests E2E)                     │
│  ┌─────────────────────────────────────────┐    │
│  │  Interface Adapters                     │    │
│  │  (Controllers, Presenters, Gateways)    │    │
│  │  ┌───────────────────────────────┐      │    │
│  │  │  Application Business Rules   │      │    │
│  │  │  (Use Cases)                  │      │    │
│  │  │  ┌─────────────────────────┐  │      │    │
│  │  │  │  Enterprise Business    │  │      │    │
│  │  │  │  Rules (Entities)       │  │      │    │  Couche interne
│  │  │  └─────────────────────────┘  │      │    │
│  │  └───────────────────────────────┘      │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
          Toutes les flèches pointent vers l'intérieur
```

**Entities (centre) :** règles métier d'entreprise — les Aggregates, Value Objects
et Domain Services du DDD. Elles ne connaissent aucune couche externe.

**Use Cases :** orchestration des cas d'usage — les Application Services.
Ils connaissent les Entities et définissent les interfaces des Adapters (Ports).

**Interface Adapters :** traduction entre le monde externe et les Use Cases —
controllers HTTP, presenters, repository implementations.

**Frameworks & Drivers :** détails d'infrastructure — Flask, Django, SQLAlchemy,
Stripe SDK. Ils sont à l'extérieur et peuvent être remplacés sans toucher aux
couches internes.

---

### 7.3 Hexagonale vs Onion vs Clean — comparaison pratique

Ces trois architectures sont souvent présentées comme distinctes. En pratique,
leurs différences sont mineures et leur esprit est identique.

| Dimension | Hexagonale (Cockburn) | Onion (Palermo) | Clean (Martin) |
|---|---|---|---|
| Métaphore | Hexagone avec faces | Pelure d'oignon | Cercles concentriques |
| Vocabulaire | Ports / Adapters | Layers / Interfaces | Entities / Use Cases / Adapters |
| Règle centrale | Tout passe par un Port | Dépendances vers l'intérieur | Règle de dépendance |
| Granularité | Deux zones (intérieur / extérieur) | Quatre couches nommées | Quatre couches nommées |
| Point fort | Testabilité, pilotage multiple | Explicite sur le rôle des couches | Synthétique, vocabulaire universel |

**Conseil pratique :** choisissez le vocabulaire que votre équipe comprend le
mieux. Les trois architectures produisent la même structure de code si on les
applique rigoureusement. Ne perdez pas de temps à débattre des noms — investissez-le
dans le respect de la règle de dépendance.

---

### 7.4 CQRS — Command Query Responsibility Segregation

#### Principe

Le CQRS (Bertrand Meyer, popularisé par Greg Young) sépare les opérations
d'écriture (*Commands*) des opérations de lecture (*Queries*) en deux modèles
distincts.

**CQS (Command Query Separation) :** principe de base formulé par Meyer —
une méthode ne doit pas à la fois modifier l'état *et* retourner une valeur.
Soit elle modifie (*Command*), soit elle lit (*Query*), jamais les deux.

**CQRS** applique ce principe à l'échelle de l'architecture : deux modèles
séparés, potentiellement deux bases de données distinctes.

```
                    ┌─────────────────────────────────┐
                    │         WRITE SIDE               │
  Command  ────────►│  Command Handler                 │
  (PlaceOrder)      │  → Aggregate                     │
                    │  → Repository (écriture)         │
                    │  → Domain Events publiés         │
                    └──────────────┬──────────────────┘
                                   │ Domain Events
                                   ▼
                    ┌─────────────────────────────────┐
                    │       PROJECTION (Sync)          │
                    │  Event Handler met à jour        │
                    │  le Read Model                   │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │         READ SIDE                │
  Query    ────────►│  Query Handler                   │
  (GetOrderSummary) │  → Read Model (optimisé lecture) │
                    │  → Retourne un DTO               │
                    └─────────────────────────────────┘
```

#### Implémentation

```php
<?php

declare(strict_types=1);

// ── Write side ────────────────────────────────────────────────────────────────

final class PlaceOrderCommand
{
    public function __construct(
        public readonly string $customerId,
        public readonly string $productId,
        public readonly int    $quantity,
    ) {}
}

final class PlaceOrderHandler
{
    public function __construct(
        private readonly OrderRepository    $orders,
        private readonly DomainEventBus     $eventBus,
    ) {}

    public function handle(PlaceOrderCommand $command): OrderId
    {
        $order = Order::create(
            new CustomerId($command->customerId),
            new ProductId($command->productId),
            $command->quantity,
        );

        $this->orders->save($order);

        foreach ($order->pullDomainEvents() as $event) {
            $this->eventBus->publish($event);
        }

        return $order->id();
    }
}


// ── Read side ─────────────────────────────────────────────────────────────────

final class GetOrderSummaryQuery
{
    public function __construct(public readonly string $orderId) {}
}

final class OrderSummaryDto
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $customerName,
        public readonly string $status,
        public readonly string $totalFormatted,
        public readonly string $createdAt,
    ) {}
}

final class GetOrderSummaryHandler
{
    public function __construct(private readonly \PDO $readDb) {}

    public function handle(GetOrderSummaryQuery $query): ?OrderSummaryDto
    {
        // Le Read Model est une vue dénormalisée optimisée pour la lecture.
        // Pas d'Aggregate, pas de règles métier — seulement une projection plate.
        $row = $this->readDb->prepare(
            'SELECT o.id, c.name, o.status, o.total_cents, o.created_at
             FROM order_summaries o
             JOIN customers c ON o.customer_id = c.id
             WHERE o.id = :id'
        );
        $row->execute([':id' => $query->orderId]);
        $data = $row->fetch(\PDO::FETCH_ASSOC);

        if (!$data) return null;

        return new OrderSummaryDto(
            orderId:        $data['id'],
            customerName:   $data['name'],
            status:         $data['status'],
            totalFormatted: number_format($data['total_cents'] / 100, 2) . ' €',
            createdAt:      $data['created_at'],
        );
    }
}


// ── Projection : synchronisation Read Model depuis les Domain Events ───────────

final class OrderSummaryProjection
{
    public function __construct(private readonly \PDO $readDb) {}

    public function onOrderCreated(OrderCreated $event): void
    {
        $this->readDb->prepare(
            'INSERT INTO order_summaries (id, customer_id, status, total_cents, created_at)
             VALUES (:id, :cid, :status, :total, :created_at)'
        )->execute([
            ':id'         => $event->orderId->value,
            ':cid'        => $event->customerId->value,
            ':status'     => 'PENDING',
            ':total'      => $event->totalCents,
            ':created_at' => $event->occurredAt->format('Y-m-d H:i:s'),
        ]);
    }

    public function onOrderConfirmed(OrderConfirmed $event): void
    {
        $this->readDb->prepare(
            'UPDATE order_summaries SET status = :status WHERE id = :id'
        )->execute([':status' => 'CONFIRMED', ':id' => $event->orderId->value]);
    }
}
```

**Quand appliquer le CQRS ?**

Le CQRS n'est pas adapté à tous les contextes. Il ajoute une complexité réelle
(deux modèles, synchronisation, éventuelle consistance). Il se justifie quand :

- les profils de charge en lecture et en écriture sont très différents
  (ex. : 1 écriture pour 1 000 lectures) ;
- le modèle de lecture optimal est très différent du modèle d'écriture
  (jointures complexes, agrégations) ;
- on souhaite que la lecture ne soit jamais bloquée par les transactions d'écriture.

---

### 7.5 Event Sourcing — introduction

#### Principe

Dans un système classique, on persiste l'*état courant* d'un agrégat
(une ligne dans une table). Dans un système à Event Sourcing, on persiste la
*séquence des événements* qui ont conduit à cet état. L'état courant est
*reconstitué* en rejouant les événements depuis le début.

```
Approche classique :
  Order { id: 1, status: "CONFIRMED", total: 89.97, updated_at: ... }

Event Sourcing :
  OrderCreated    { order_id: 1, product_id: 42, quantity: 3, at: T1 }
  DiscountApplied { order_id: 1, promo_code: "SUMMER", amount: 10.00, at: T2 }
  OrderConfirmed  { order_id: 1, transaction_id: "TX-001", at: T3 }
  ↓
  Reconstitution : rejouer T1 + T2 + T3 → même état que la BDD classique
```

#### Implémentation minimale

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any


# ── Event Store (interface) ───────────────────────────────────────────────────

class EventStorePort(ABC):

    @abstractmethod
    def append(self, aggregate_id: str, events: list, expected_version: int) -> None:
        """
        Persiste les événements.
        expected_version : version attendue avant l'ajout — permet la détection
        de conflits d'écriture concurrente (optimistic concurrency).
        """
        ...

    @abstractmethod
    def load(self, aggregate_id: str) -> list:
        """Charge tous les événements d'un agrégat dans l'ordre chronologique."""
        ...


# ── Aggregate reconstitué depuis les événements ───────────────────────────────

class EventSourcedOrder:
    """
    Aggregate dont l'état est reconstitué en rejouant les Domain Events.
    """

    def __init__(self):
        self._id:        str | None     = None
        self._status:    str            = "NONE"
        self._total:     float          = 0.0
        self._version:   int            = 0
        self._new_events: list          = []

    @classmethod
    def create(cls, order_id: str, product_id: str, unit_price: float, quantity: int):
        order = cls()
        event = OrderCreated(order_id, product_id, unit_price, quantity, datetime.utcnow())
        order._apply(event)
        order._new_events.append(event)
        return order

    def confirm(self, transaction_id: str) -> None:
        if self._status != "PENDING":
            raise ValueError("Seules les commandes en attente peuvent être confirmées.")
        event = OrderConfirmed(self._id, transaction_id, datetime.utcnow())
        self._apply(event)
        self._new_events.append(event)

    # ── Application des événements (pure : pas d'effet de bord) ──────────────

    def _apply(self, event) -> None:
        """Dispatche vers le bon handler d'événement."""
        handler_name = f"_on_{type(event).__name__}"
        handler = getattr(self, handler_name, None)
        if handler:
            handler(event)
        self._version += 1

    def _on_OrderCreated(self, event) -> None:
        self._id     = event.order_id
        self._status = "PENDING"
        self._total  = event.unit_price * event.quantity

    def _on_OrderConfirmed(self, event) -> None:
        self._status = "CONFIRMED"

    # ── Reconstitution depuis l'Event Store ───────────────────────────────────

    @classmethod
    def reconstitute(cls, events: list) -> "EventSourcedOrder":
        """Rejoue tous les événements pour reconstruire l'état courant."""
        order = cls()
        for event in events:
            order._apply(event)
        order._new_events = []  # événements chargés ≠ nouveaux événements
        return order

    def pull_new_events(self) -> list:
        events, self._new_events = self._new_events, []
        return events

    @property
    def version(self) -> int: return self._version


# ── Utilisation ───────────────────────────────────────────────────────────────

class OrderRepository:
    def __init__(self, event_store: EventStorePort):
        self._store = event_store

    def save(self, order: EventSourcedOrder) -> None:
        new_events = order.pull_new_events()
        self._store.append(
            aggregate_id=order._id,
            events=new_events,
            expected_version=order.version - len(new_events),
        )

    def find_by_id(self, order_id: str) -> EventSourcedOrder | None:
        events = self._store.load(order_id)
        if not events:
            return None
        return EventSourcedOrder.reconstitute(events)
```

#### Avantages et coûts de l'Event Sourcing

**Avantages :**

- **Audit log complet et gratuit :** l'Event Store est un journal immuable de
  tout ce qui s'est passé dans le système.
- **Voyage dans le temps :** on peut reconstituer l'état de n'importe quel
  agrégat à n'importe quel moment passé.
- **Base naturelle du CQRS :** les projections Read Model sont construites en
  consommant les événements.
- **Découplage temporel :** de nouveaux consommateurs d'événements (nouvelles
  projections, nouveaux Read Models) peuvent être ajoutés après coup et
  reconstitués depuis le début.

**Coûts :**

- **Complexité cognitive élevée :** le modèle mental est très différent du CRUD.
  Le temps d'apprentissage de l'équipe est significatif.
- **Gestion des migrations d'événements :** si la structure d'un événement change,
  il faut versionner les événements et gérer les anciens formats (*event upcasting*).
- **Performance de reconstitution :** pour les agrégats avec des milliers
  d'événements, rejouer depuis le début est lent → nécessite des *snapshots*.
- **Testing plus complexe :** les tests d'intégration nécessitent un Event Store.

**Quand l'adopter :**

L'Event Sourcing ne se justifie que si au moins deux des conditions suivantes
sont vraies :
- L'audit log complet est une exigence fonctionnelle (finance, santé, juridique).
- La reconstitution temporelle est nécessaire (replay de scénarios, correction
  rétroactive d'erreurs).
- Plusieurs Bounded Contexts consomment les mêmes événements et doivent être
  synchronisés de façon asynchrone.

Pour la grande majorité des applications métier, une BDD relationnelle classique
avec une table d'audit suffit largement.

---

## Session 8 — Débat structuré : pragmatisme vs purisme (15 h 15 – 16 h 30)

### 8.1 Etude de cas — deux contextes, deux approches

#### Cas A : Start-up en phase d'amorçage

**Contexte :**
- Équipe de 3 développeurs, 6 mois pour valider le marché
- Budget serveur limité, pas encore de clients payants
- Le domaine métier n'est pas encore stabilisé (les règles changent chaque semaine)
- Priorité absolue : livrer des fonctionnalités pour apprendre du marché

**Approche recommandée :**

```python
# Un seul fichier, une seule table, des routes Flask simples
# Pas d'Aggregate, pas de CQRS, pas d'Event Sourcing

@app.route("/orders", methods=["POST"])
def create_order():
    data = request.json
    # Validation minimale
    if not data.get("product_id") or data.get("quantity", 0) <= 0:
        return {"error": "Invalid data"}, 400

    order_id = db.execute(
        "INSERT INTO orders (customer_id, product_id, quantity, status) "
        "VALUES (?, ?, ?, 'pending')",
        (data["customer_id"], data["product_id"], data["quantity"])
    ).lastrowid
    db.commit()
    return {"order_id": order_id}, 201
```

**Pourquoi c'est juste** : YAGNI (You Aren't Gonna Need It). Les Aggregates,
le CQRS et l'Event Sourcing ont un coût de mise en place et de maintenance. Si
le produit échoue dans 3 mois (ce qui est statistiquement probable), ce coût
est pure perte. Si le produit réussit, les 6 mois suivants permettront de
refactoriser en connaissance de cause — le domaine sera alors stabilisé.

**Ce qu'il faut quand même faire :** des tests (même basiques), du nommage
correct, une séparation minimale entre lecture et écriture dans les fonctions.
La dette technique *inconsciente* s'accumule même dans une start-up.

---

#### Cas B : Système bancaire de paiements

**Contexte :**
- Équipe de 20 développeurs, système en production depuis 5 ans
- Exigences réglementaires strictes (audit, traçabilité complète)
- 50 000 transactions par jour, pics à 500 transactions par seconde
- Le domaine est stable et bien compris ; les règles métier sont codifiées
- Disponibilité 99,99 % requise contractuellement

**Approche recommandée :**
- DDD tactique complet sur le Core Domain (paiements, compensations, réconciliation)
- CQRS pour séparer les flux transactionnels des rapports réglementaires
- Event Sourcing sur les agrégats critiques (audit trail obligatoire)
- Architecture hexagonale pour l'isolation totale du domaine
- Tests d'architecture automatisés (aucune violation tolérée)

**Pourquoi c'est juste** : la complexité technique est justifiée par la complexité
métier et les exigences non fonctionnelles. Un audit trail manquant peut entraîner
une amende réglementaire. La disponibilité requiert une séparation lecture/écriture.
Le système vivra 10 ans : investir dans la maintenabilité est rentable.

---

### 8.2 YAGNI vs conception anticipatrice

**YAGNI** (You Aren't Gonna Need It, formulé par Ron Jeffries dans XP) est souvent
cité comme antidote à la sur-ingénierie. Son sens précis est : *n'implémentez
pas une fonctionnalité tant qu'elle n'est pas nécessaire*. Il ne dit pas :
*ne concevez pas bien*.

La tension réelle n'est pas entre YAGNI et bonne conception — c'est entre
**spéculation** et **certitude** :

| Décision | Spéculative (évitez) | Certaine (faites) |
|---|---|---|
| Abstraire une dépendance | Si une seule implémentation existera jamais | Si les tests nécessitent un double |
| Ajouter un Cache | Parce que « ça pourrait être lent » | Parce que les métriques montrent un goulot |
| Introduire CQRS | Parce que c'est « mieux » | Parce que les lectures bloquent les écritures en production |
| Créer un Value Object | Parce que « un jour peut-être » | Parce qu'un bug de validation existe déjà |
| Ecrire des tests | Les tests « prennent du temps » | Toujours : ils évitent les régressions certaines |

La règle des **deux instances** (vue lors du DIP) s'applique ici : n'abstraire
que ce qu'on a déjà vu varier au moins une fois. La première fois, on code en
dur ; la deuxième, on abstrait.

---

### 8.3 Architecture Decision Records (ADR)

Un ADR est un document court qui capture une décision architecturale importante :
le contexte qui l'a motivée, la décision elle-même, et ses conséquences.

Proposé par Michael Nygard (2011), l'ADR répond à un problème fréquent : six mois
après une décision, plus personne ne se souvient pourquoi elle a été prise.
Les nouveaux arrivants remettent en question des choix qui avaient d'excellentes
raisons d'être — ou, pire, les anciens membres remettent en question des choix
dont ils ont oublié les contraintes.

#### Format standard (Nygard)

```markdown
# ADR-0012 : Adoption de l'architecture hexagonale pour le module Paiements

## Statut
Accepté — 2024-03-15

## Contexte
Le module Paiements est actuellement couplé directement à Stripe via des appels SDK
dans les contrôleurs. Cela rend impossible :
- les tests unitaires du flux de paiement sans appel réseau réel ;
- le changement de prestataire PSP sans réécriture du module ;
- le test du comportement en cas de refus de carte sans carte de test Stripe.

Trois fournisseurs PSP sont envisagés pour 2025 (Stripe, Adyen, Payplug).

## Décision
Adopter l'architecture hexagonale pour le module Paiements :
- Définir un Port `PaymentGatewayPort` dans la couche domaine.
- Implémenter un Adapter `StripePaymentAdapter` dans la couche infrastructure.
- Tous les tests utiliseront `FakePaymentGateway` implémentant `PaymentGatewayPort`.

## Conséquences

Positives :
- Tests unitaires du flux de paiement sans dépendance Stripe.
- Changement de PSP = nouveau Adapter uniquement, sans toucher au domaine.
- Simuler les refus, timeouts, erreurs 3DS dans les tests.

Négatives :
- Une couche d'indirection supplémentaire (Port + Adapter vs appel direct).
- Les développeurs junior devront comprendre le pattern avant de contribuer.
- Migration du code existant estimée à 3 jours de développement.

## Alternatives considérées
- Mocker le SDK Stripe directement : rejeté car crée un couplage aux détails
  du SDK (si Stripe change son API, les mocks cassent aussi).
- Garder le couplage direct : rejeté car bloque les tests et le changement de PSP.
```

**Bonnes pratiques ADR :**

- Un ADR par décision significative (pas par ligne de code).
- Les stocker dans le dépôt Git, au même niveau que le code (dossier `docs/adr/`).
- Ne jamais modifier un ADR accepté : créer un nouvel ADR qui le supersède.
- Numéroter séquentiellement pour conserver l'histoire.
- Le statut peut être : *Proposé*, *Accepté*, *Déprécié*, *Supersédé par ADR-XXXX*.

---

### 8.4 La revue de code comme outil pédagogique et culturel

La revue de code (*code review*) est souvent perçue comme un mécanisme de
contrôle qualité. C'est aussi — et peut-être surtout — un outil de transmission
de connaissance et de construction d'une culture technique commune.

#### Ce que la revue de code apporte au-delà des bugs

**Transmission du langage partagé :** si un développeur utilise `userList` au
lieu de `users`, la revue est le bon endroit pour aligner sur l'Ubiquitous
Language du projet.

**Propagation des patterns :** un nouveau membre de l'équipe apprend le pattern
Repository en voyant son usage commenté en revue, pas en lisant une documentation.

**Détection des violations d'architecture :** un contrôleur qui accède directement
à la BDD, un Aggregate qui dépend d'une bibliothèque HTTP — autant de violations
détectables en revue avant qu'elles ne s'accumulent.

**Culture du feedback constructif :** une revue bien menée crée un espace sûr
pour questionner les choix sans attaquer les personnes.

#### Principes d'une revue de code efficace

```
Pour le reviewer :
  - Questionner plutôt qu'affirmer : « Avez-vous envisagé X ? » plutôt que « Il faut faire X. »
  - Distinguer ce qui est bloquant (bug, violation d'architecture) de ce qui est
    une suggestion (style, alternative possible).
  - Expliquer le pourquoi : « Ce couplage rendra le test difficile parce que... »
  - Chercher à comprendre avant de critiquer : peut-être y a-t-il une contrainte
    non documentée.

Pour l'auteur :
  - Ne pas prendre les commentaires personnellement : c'est le code qui est revu,
    pas la personne.
  - Répondre à chaque commentaire, même pour dire « je ne suis pas d'accord, parce que... »
  - Créer des PRs petites et fréquentes plutôt que des monstres de 2 000 lignes.

Pour l'équipe :
  - Définir et documenter les standards une fois pour toutes (linter, formatter,
    conventions de nommage) pour ne pas perdre la revue en discussions de style.
  - Automatiser ce qui peut l'être (CI, tests, analyse statique) : la revue humaine
    doit porter sur la conception, pas sur la syntaxe.
```

---

## Session 9 — Synthèse & ressources pour aller plus loin (16 h 30 – 17 h)

### 9.1 Carte des concepts et de leurs liens

La carte suivante positionne tous les concepts abordés sur deux axes :
niveau de granularité (micro → macro) et nature (structure → comportement).

```
MACRO ─────────────────────────────────────────────────────────────────────────
│
│  Event Sourcing ─────────► CQRS ──────────────────► Architecture hexagonale
│       │                      │                              │
│       │                      │                              │
│  Domain Events ──────────► Bounded Context ─────────────► Clean Architecture
│       │                      │
│       │                      ▼
│  Event Storming ─────────► DDD (Ubiquitous Language, Aggregates, Repositories)
│       │                      │
│       │                      ▼
│  Refactoring ─────────────► SOLID ────────────────────────► Design Patterns
│  (Mikado, HOF)               │
│                              ▼
│  Clean Code ─────────────► Nommage, fonctions, smells
│
│  PBT, Tests d'architecture ► TDD ─────────────────────────► Test doubles
│
│  Programmation fonctionnelle : immutabilité, Maybe, Either, Railway
│
MICRO ─────────────────────────────────────────────────────────────────────────
         STRUCTURE                                      COMPORTEMENT
```

**Liens clés à retenir :**

- La **règle de dépendance** (Clean Architecture) = le **DIP** (SOLID) à l'échelle
  du système.
- Le **Repository** DDD = le **DIP** appliqué à la persistence.
- Les **Value Objects** DDD = l'élimination de la **Primitive Obsession** (smells)
  avec des **fonctions pures** (FP).
- Les **Domain Events** DDD = la base naturelle du **CQRS** et de l'**Event Sourcing**.
- L'**Ubiquitous Language** = le **nommage expressif** (Clean Code) à l'échelle du
  système.
- Le **TDD** = le test de la **testabilité** de l'architecture (si un test est difficile
  à écrire, l'architecture est couplée).
- Les **tests d'architecture** = la vérification automatique de la **règle de dépendance**.
- L'**ADR** = la documentation des choix qui justifient YAGNI ou sa transgression.

---

### 9.2 Grille de décision — quelle pratique pour quel contexte ?

Cette grille aide à choisir le niveau d'investissement dans chaque pratique
en fonction des caractéristiques du projet.

| Pratique | Toujours | Si complexité métier | Si haute disponibilité | Si audit requis |
|---|:---:|:---:|:---:|:---:|
| Clean Code & nommage | Oui | Oui | Oui | Oui |
| Tests unitaires | Oui | Oui | Oui | Oui |
| SOLID | Oui | Oui | Oui | Oui |
| Value Objects | Recommandé | Oui | Oui | Oui |
| DDD stratégique | Recommandé | Oui | Oui | Oui |
| DDD tactique complet | Non | Oui | Partiel | Oui |
| Architecture hexagonale | Recommandé | Oui | Oui | Oui |
| TDD | Recommandé | Oui | Oui | Oui |
| Property-based testing | Optionnel | Oui | Oui | Recommandé |
| CQRS | Non | Partiel | Oui | Recommandé |
| Event Sourcing | Non | Non | Non | Oui |
| ADR | Recommandé | Oui | Oui | Oui |

---

### 9.3 Katas recommandés pour s'entraîner

Les katas sont des exercices de code à pratiquer et re-pratiquer pour développer
des automatismes. Ils sont courts (1-3 heures), bien définis, et s'améliorent
à chaque itération.

#### Katas de base (Clean Code, TDD)

**String Calculator** *(exercice du TD de ce matin)*  
Excellent pour pratiquer le cycle Red/Green/Refactor depuis zéro.  
Référence : Roy Osherove — https://osherove.com/tdd-kata-1

**FizzBuzz** *(version étendue vue ce matin)*  
Idéal pour pratiquer OCP et la progression par refactoring.

**Roman Numerals**  
Convertir des entiers en chiffres romains et vice-versa. Très utile pour
pratiquer le TDD sur une logique discrète.

#### Katas de refactoring (Legacy Code)

**Gilded Rose** *(le kata de refactoring le plus célèbre)*  
Une fonction `update_quality()` de 60 lignes d'imbrications catastrophiques
à refactoriser sans casser les comportements existants. Disponible en PHP et Python.  
Référence : https://github.com/emilybache/GildedRose-Refactoring-Kata

```python
# L'état initial de Gilded Rose — reconnaissez-vous les smells ?
def update_quality(items):
    for item in items:
        if item.name != "Aged Brie" and item.name != "Backstage passes to a TAFKAL80ETC concert":
            if item.quality > 0:
                if item.name != "Sulfuras, Hand of Ragnaros":
                    item.quality = item.quality - 1
        else:
            if item.quality < 50:
                item.quality = item.quality + 1
                if item.name == "Backstage passes to a TAFKAL80ETC concert":
                    if item.sell_in < 11:
                        if item.quality < 50:
                            item.quality = item.quality + 1
                    if item.sell_in < 6:
                        if item.quality < 50:
                            item.quality = item.quality + 1
        # ... et ça continue ainsi pendant 50 lignes
```

**Trivia** (*Working Effectively with Legacy Code*)  
Refactoriser un jeu de questions-réponses sans tests existants. Oblige à écrire
les tests de caractérisation avant de pouvoir refactoriser.  
Référence : https://github.com/jbrains/trivia

**Tennis Refactoring Kata**  
Cinq implémentations différentes (toutes fonctionnelles mais mal structurées)
d'un système de score de tennis à refactoriser.

#### Katas de conception DDD

**Bank Account Kata**  
Modéliser un compte bancaire avec dépôt, retrait, découvert autorisé, impression
de relevé. Introduit naturellement les Aggregates, Value Objects (Money) et
Domain Events.

```
Exigences :
  - Dépôt et retrait avec montant et date
  - Impression du relevé : date | montant | solde
  - Le solde ne peut pas être négatif (invariant de l'agrégat)
  - Bonus : ajouter les Domain Events et une projection de lecture
```

**Bowling Game Kata** *(Kent Beck)*  
Calculer le score d'une partie de bowling. Connu pour les designs qui semblent
évoluer naturellement pendant le TDD — chaque étape révèle des patterns.

---

### 9.4 Ressources pour aller plus loin

#### Livres fondamentaux (dans l'ordre de lecture recommandé)

1. **Martin, R. C.** — *Clean Code* (2008, Prentice Hall)  
   Premier livre à lire pour tout développeur sérieux. Dense, parfois controversé
   sur certains points, mais essentiel sur le nommage et la structure.

2. **Martin, R. C.** — *Agile Software Development, Principles, Patterns, and
   Practices* (2002, Prentice Hall)  
   La source originale des principes SOLID, avec des exemples complets.

3. **Fowler, M.** — *Refactoring: Improving the Design of Existing Code*, 2e éd.
   (2018, Addison-Wesley)  
   Le catalogue de référence des refactorisations. A lire avec un IDE ouvert.

4. **Evans, E.** — *Domain-Driven Design: Tackling Complexity in the Heart of
   Software* (2003, Addison-Wesley)  
   Le livre fondateur du DDD. Dense et riche ; prévoir plusieurs lectures.

5. **Vernon, V.** — *Implementing Domain-Driven Design* (2013, Addison-Wesley)  
   Complément pratique indispensable d'Evans. Beaucoup de code, très concret.

6. **Martin, R. C.** — *Clean Architecture* (2017, Prentice Hall)  
   Synthèse des architectures hexagonale, Onion et Clean. Court et accessible.

7. **Feathers, M.** — *Working Effectively with Legacy Code* (2004, Prentice Hall)  
   Techniques pour introduire les tests et le découplage dans du code sans tests.
   Le compagnon indispensable du Gilded Rose kata.

8. **Freeman, S. & Pryce, N.** — *Growing Object-Oriented Software, Guided by
   Tests* (2009, Addison-Wesley)  
   Le meilleur livre sur l'articulation entre TDD et conception OO. Montre
   comment le TDD fait émerger une architecture hexagonale naturellement.

9. **Wlaschin, S.** — *Domain Modeling Made Functional* (2018, Pragmatic
   Bookshelf)  
   DDD et programmation fonctionnelle en F#. Même si vous ne pratiquez pas F#,
   les concepts sont directement transposables à Python ou TypeScript.

#### Articles et ressources en ligne

- **Cockburn, A.** (2005). *Hexagonal Architecture.*  
  https://alistair.cockburn.us/hexagonal-architecture/  
  L'article original, court et précis.

- **Fowler, M.** (2011). *CQRS.*  
  https://martinfowler.com/bliki/CQRS.html  
  Explication concise avec mise en garde sur la complexité.

- **Brandolini, A.** Blog : *ziobrando.blogspot.com*  
  Articles sur l'Event Storming par son inventeur.

- **Wlaschin, S.** — *F# for Fun and Profit* : https://fsharpforfun.com  
  La série d'articles sur le Railway-Oriented Programming et le DDD fonctionnel.

- **Martin, R. C.** — *The Clean Code Blog* : https://blog.cleancoder.com  
  Articles sur l'architecture, les principes et le craftsmanship.

- **Nygard, M.** (2011). *Documenting Architecture Decisions.*  
  https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions  
  L'article original sur les ADR.

#### Communautés et événements

**Software Craftsmanship** (https://manifesto.softwarecraftsmanship.org)  
Le mouvement qui prolonge l'Agile Manifesto sur les aspects techniques.
Groupes locaux dans la plupart des grandes villes françaises.

**DDD Europe** (https://ddd-europe.org)  
Conférence annuelle dédiée au Domain-Driven Design. Nombreuses vidéos
disponibles gratuitement sur YouTube.

**DDD Community** (https://dddcommunity.org)  
Ressources, conférences et groupes de travail autour du DDD.

**Meetups locaux**  
Cherchez « Software Craftsmanship », « DDD », « Clean Code » sur Meetup.com
dans votre ville. Ces communautés pratiquent régulièrement des katas en groupe
— l'un des meilleurs moyens d'apprendre.

---

### 9.5 Ce que vous avez appris — et ce qui reste à faire

#### Ce que cette formation a posé

En deux jours, vous avez parcouru un chemin qui va du plus petit détail
(le nom d'une variable) jusqu'à la grande échelle d'un système distribué
(Event Sourcing, CQRS). Les concepts forment un tout cohérent :

- **Lisibilité** (Clean Code) → les autres développeurs comprennent votre intention.
- **Testabilité** (TDD, DIP) → vous pouvez modifier le code sans craindre les régressions.
- **Extensibilité** (OCP, DDD) → le code peut évoluer avec les besoins métier.
- **Maintenabilité** (SRP, refactoring) → la dette technique reste sous contrôle.
- **Scalabilité conceptuelle** (CQRS, hexagonale) → l'architecture tient quand
  le système grandit.

#### Ce qui ne s'apprend que par la pratique

Connaître ces concepts est nécessaire ; les maîtriser demande des mois ou des
années de pratique délibérée. Trois recommandations pour la suite :

**1. Pratiquez les katas régulièrement**  
Recommencez le Gilded Rose Kata dans deux semaines. Vous verrez les smells plus
vite, trouverez le refactoring plus fluide. Répétez jusqu'à ce que le cycle
Red/Green/Refactor devienne un réflexe.

**2. Relisez du code**  
Passez 15 minutes par semaine à lire du code open source de qualité : les sources
de Django, de Symfony, d'un projet DDD exemplaire sur GitHub. La lecture de bon
code est aussi formatrice que l'écriture.

**3. Introduisez progressivement**  
Ne tentez pas de tout appliquer d'un coup sur votre prochain projet. Choisissez
une pratique à la fois : cette semaine, nommez mieux ; le mois prochain, introduisez
les Value Objects là où la Primitive Obsession est évidente ; dans trois mois,
essayez TDD sur un nouveau module.

Le softwarecraftsmanship est une pratique, pas une certification.

---

## Récapitulatif de la formation complète

| Demi-journée | Thème | Concepts clés |
|---|---|---|
| Jour 1 matin | Clean Code & Refactoring | Nommage, fonctions, LoD, POLA, dette technique, smells, Mikado |
| Jour 1 après-midi | DDD & Event Storming | Ubiquitous Language, Entities, Value Objects, Aggregates, Repositories, Bounded Contexts, Event Storming |
| Jour 2 matin | FP & TDD | Immutabilité, fonctions pures, Maybe, Either, Railway, TDD, test doubles, PBT, tests d'architecture |
| Jour 2 après-midi | Architecture évolutive & pragmatisme | Hexagonale, Clean Architecture, CQRS, Event Sourcing, YAGNI, ADR, code review |

**Fil conducteur :** chaque concept répond à la même question, posée à un niveau
différent — *comment écrire du logiciel qui reste lisible, testable et modifiable
au fil du temps, sans surprise ?*
