# SRP — Corrigés des exercices niveaux 2 et 3

Ces corrigés proposent **une** solution parmi d'autres possibles. L'objectif n'est pas d'imposer une architecture unique, mais d'illustrer les raisonnements qui mènent à un découpage cohérent avec le principe de responsabilité unique.

---

## Exercice 2 (niveau intermédiaire) — Découpage de `OrderProcessor` en Python

### Rappel de l'énoncé

La classe `OrderProcessor` cumule cinq responsabilités :
1. Validation des données de la commande (montant positif, produit existant) ;
2. Application d'une réduction selon un code promo ;
3. Insertion de la commande en base SQLite ;
4. Génération d'une facture PDF via une bibliothèque externe ;
5. Envoi de la facture par e-mail au client.

### Identification des acteurs (au sens SRP)

Avant de découper, il faut identifier *qui* est à l'origine de chaque changement possible :

| Responsabilité | Acteur dont les règles peuvent changer |
|---|---|
| Validation | L'équipe métier (règles de gestion des commandes) |
| Calcul de remise | L'équipe marketing (politique tarifaire) |
| Persistance | L'équipe infrastructure (choix de la base de données) |
| Génération PDF | L'équipe produit (mise en page, charte graphique) |
| Envoi e-mail | L'équipe infrastructure / CRM (fournisseur SMTP, template) |

Cinq acteurs distincts → cinq responsabilités → cinq classes (ou modules) distincts.

### Diagramme de classes textuel

```
OrderValidator
  + validate(order_data: dict) -> None   # lève ValueError si invalide

DiscountCalculator
  + apply(price: float, promo_code: str) -> float

OrderRepository
  + save(order: Order) -> int            # retourne l'id inséré

InvoiceGenerator
  + generate(order: Order) -> bytes      # retourne le PDF en binaire

InvoiceMailer
  + send(recipient_email: str, invoice_pdf: bytes) -> None

OrderApplicationService                 # orchestrateur (service applicatif)
  - validator:   OrderValidator
  - discounter:  DiscountCalculator
  - repository:  OrderRepository
  - generator:   InvoiceGenerator
  - mailer:      InvoiceMailer
  + process(order_data: dict) -> int
```

L'objet valeur `Order` transporte les données entre les classes sans logique propre.

### Implémentation

```python
# ── order.py ────────────────────────────────────────────────────────────────

from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class OrderLine:
    product_id: int
    quantity: int
    unit_price: float


@dataclass
class Order:
    customer_email: str
    lines: list[OrderLine]
    promo_code: str = ""
    total: float = 0.0
    created_at: datetime = field(default_factory=datetime.utcnow)
    id: int | None = None


# ── validation.py ────────────────────────────────────────────────────────────

class OrderValidator:
    """
    Responsabilité unique : vérifier qu'une commande satisfait les règles métier
    avant tout traitement.
    Raison de changer : l'équipe métier modifie les règles de validation.
    """

    def __init__(self, product_catalog: set[int]):
        # Le catalogue des produits valides est injecté : pas de dépendance BDD ici.
        self._catalog = product_catalog

    def validate(self, order: Order) -> None:
        if not order.customer_email or "@" not in order.customer_email:
            raise ValueError("Adresse e-mail client invalide.")
        if not order.lines:
            raise ValueError("Une commande doit contenir au moins une ligne.")
        for line in order.lines:
            if line.quantity <= 0:
                raise ValueError(f"Quantité invalide pour le produit {line.product_id}.")
            if line.unit_price < 0:
                raise ValueError(f"Prix négatif pour le produit {line.product_id}.")
            if line.product_id not in self._catalog:
                raise ValueError(f"Produit inconnu : {line.product_id}.")


# ── discount.py ──────────────────────────────────────────────────────────────

class DiscountCalculator:
    """
    Responsabilité unique : calculer le montant final après application
    d'un éventuel code promotionnel.
    Raison de changer : l'équipe marketing modifie la politique de remise.
    """

    # Table des remises (pourcentage). En production, viendrait d'une BDD ou
    # d'un fichier de configuration.
    _PROMO_RATES: dict[str, float] = {
        "BIENVENUE10": 0.10,
        "ETE20":       0.20,
        "VIP50":       0.50,
    }

    def apply(self, base_price: float, promo_code: str) -> float:
        rate = self._PROMO_RATES.get(promo_code.upper(), 0.0)
        return round(base_price * (1 - rate), 2)


# ── repository.py ────────────────────────────────────────────────────────────

import sqlite3
from abc import ABC, abstractmethod


class OrderRepositoryInterface(ABC):
    """
    Abstraction définie par la couche métier (anticipation du DIP).
    Permet de substituer SQLite par n'importe quel autre moteur sans
    toucher au service applicatif.
    """

    @abstractmethod
    def save(self, order: Order) -> int:
        ...


class SqliteOrderRepository(OrderRepositoryInterface):
    """
    Responsabilité unique : persister et récupérer les commandes en SQLite.
    Raison de changer : l'équipe infrastructure change de moteur de BDD
    ou de schéma.
    """

    def __init__(self, db_path: str):
        self._conn = sqlite3.connect(db_path)
        self._create_schema()

    def _create_schema(self) -> None:
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id            INTEGER PRIMARY KEY AUTOINCREMENT,
                customer_email TEXT NOT NULL,
                total         REAL NOT NULL,
                promo_code    TEXT,
                created_at    TEXT NOT NULL
            )
        """)
        self._conn.commit()

    def save(self, order: Order) -> int:
        cursor = self._conn.execute(
            "INSERT INTO orders (customer_email, total, promo_code, created_at) "
            "VALUES (?, ?, ?, ?)",
            (order.customer_email, order.total,
             order.promo_code, order.created_at.isoformat()),
        )
        self._conn.commit()
        return cursor.lastrowid


# ── invoice_generator.py ─────────────────────────────────────────────────────

from abc import ABC, abstractmethod


class InvoiceGeneratorInterface(ABC):
    @abstractmethod
    def generate(self, order: Order) -> bytes:
        ...


class SimpleTextInvoiceGenerator(InvoiceGeneratorInterface):
    """
    Implémentation de démonstration (pas de dépendance à une lib PDF).
    Responsabilité unique : produire le contenu de la facture.
    Raison de changer : l'équipe produit modifie la mise en page ou le format.
    """

    def generate(self, order: Order) -> bytes:
        lines = [
            f"FACTURE #{order.id}",
            f"Client : {order.customer_email}",
            f"Date   : {order.created_at.strftime('%d/%m/%Y')}",
            "",
        ]
        for line in order.lines:
            lines.append(
                f"  Produit {line.product_id} x{line.quantity}"
                f" @ {line.unit_price:.2f} EUR"
            )
        lines += ["", f"TOTAL : {order.total:.2f} EUR"]
        return "\n".join(lines).encode("utf-8")


# ── mailer.py ─────────────────────────────────────────────────────────────────

import smtplib
from email.message import EmailMessage
from abc import ABC, abstractmethod


class InvoiceMailerInterface(ABC):
    @abstractmethod
    def send(self, recipient: str, invoice_pdf: bytes) -> None:
        ...


class SmtpInvoiceMailer(InvoiceMailerInterface):
    """
    Responsabilité unique : envoyer la facture par e-mail.
    Raison de changer : changement de fournisseur SMTP, de template ou
    de politique d'envoi.
    """

    def __init__(self, smtp_host: str = "localhost"):
        self._host = smtp_host

    def send(self, recipient: str, invoice_pdf: bytes) -> None:
        msg = EmailMessage()
        msg["Subject"] = "Votre facture"
        msg["From"] = "noreply@boutique.fr"
        msg["To"] = recipient
        msg.set_content("Veuillez trouver votre facture en pièce jointe.")
        msg.add_attachment(invoice_pdf, maintype="application",
                           subtype="pdf", filename="facture.pdf")
        with smtplib.SMTP(self._host) as smtp:
            smtp.send_message(msg)


# ── service.py ────────────────────────────────────────────────────────────────

class OrderApplicationService:
    """
    Service applicatif : orchestre les étapes sans contenir de logique métier.
    Responsabilité unique : coordonner le traitement d'une commande de bout en bout.
    Raison de changer : modification du *flux* de traitement (ordre des étapes,
    ajout d'une étape d'audit, etc.).

    Toutes les dépendances sont injectées -> testable unitairement.
    """

    def __init__(
        self,
        validator:  OrderValidator,
        discounter: DiscountCalculator,
        repository: OrderRepositoryInterface,
        generator:  InvoiceGeneratorInterface,
        mailer:     InvoiceMailerInterface,
    ):
        self._validator  = validator
        self._discounter = discounter
        self._repository = repository
        self._generator  = generator
        self._mailer     = mailer

    def process(self, order: Order) -> int:
        # Etape 1 — validation
        self._validator.validate(order)

        # Etape 2 — calcul du total et application de la remise
        base_total = sum(l.quantity * l.unit_price for l in order.lines)
        order.total = self._discounter.apply(base_total, order.promo_code)

        # Etape 3 — persistance
        order.id = self._repository.save(order)

        # Etape 4 — génération de la facture
        invoice_pdf = self._generator.generate(order)

        # Etape 5 — envoi par e-mail
        self._mailer.send(order.customer_email, invoice_pdf)

        return order.id
```

### Test unitaire du service applicatif

Le découpage permet de tester `OrderApplicationService` sans aucune dépendance réelle (pas de SQLite, pas de SMTP, pas de PDF).

```python
# ── test_order_service.py ─────────────────────────────────────────────────────

import pytest
from unittest.mock import MagicMock


def make_valid_order() -> Order:
    return Order(
        customer_email="alice@example.com",
        lines=[OrderLine(product_id=1, quantity=2, unit_price=49.99)],
        promo_code="BIENVENUE10",
    )


def make_service(
    catalog=None,
    repo_id=42,
    smtp_host="localhost",
):
    catalog = catalog or {1, 2, 3}
    repo = MagicMock(spec=OrderRepositoryInterface)
    repo.save.return_value = repo_id
    mailer = MagicMock(spec=InvoiceMailerInterface)
    generator = MagicMock(spec=InvoiceGeneratorInterface)
    generator.generate.return_value = b"PDF_CONTENT"
    return (
        OrderApplicationService(
            validator=OrderValidator(catalog),
            discounter=DiscountCalculator(),
            repository=repo,
            generator=generator,
            mailer=mailer,
        ),
        repo,
        mailer,
        generator,
    )


def test_process_returns_order_id():
    service, repo, _, _ = make_service(repo_id=7)
    order = make_valid_order()
    result = service.process(order)
    assert result == 7


def test_discount_applied_correctly():
    service, repo, _, _ = make_service()
    order = make_valid_order()  # 2 x 49.99 = 99.98, remise 10% -> 89.982
    service.process(order)
    assert order.total == pytest.approx(89.98, abs=0.01)


def test_invoice_sent_to_customer():
    service, _, mailer, generator = make_service()
    order = make_valid_order()
    service.process(order)
    mailer.send.assert_called_once_with("alice@example.com", b"PDF_CONTENT")


def test_invalid_product_raises():
    service, _, _, _ = make_service(catalog={99})  # produit 1 absent
    order = make_valid_order()
    with pytest.raises(ValueError, match="Produit inconnu"):
        service.process(order)
```

### Points clés à retenir

- Le service applicatif (`OrderApplicationService`) **ne contient aucune logique métier** : il se contente d'appeler les bonnes classes dans le bon ordre. Si l'ordre des étapes change, c'est la seule classe à modifier.
- Chaque classe a une **raison de changer formulable en une phrase** et attribuable à **un acteur**.
- Les interfaces (`OrderRepositoryInterface`, `InvoiceGeneratorInterface`, `InvoiceMailerInterface`) anticipent le DIP : le service applicatif ne connaît que des contrats, jamais des implémentations concrètes.

---

## Exercice 3 (niveau avancé) — Refactorisation de `PostController` en PHP

### Rappel de l'énoncé

La classe `PostController` contient six responsabilités :
1. Routage HTTP (analyse de `$_GET`/`$_POST`) ;
2. Validation des données de formulaire ;
3. Accès direct à la BDD via PDO ;
4. Rendu HTML par concaténation ;
5. Gestion du cache HTTP ;
6. Journalisation des erreurs dans un fichier.

### Étape 1 — Identification des acteurs

| Responsabilité | Acteur | Raison de changer |
|---|---|---|
| Routage HTTP | Equipe infrastructure / framework | Changement de framework, migration vers une API REST |
| Validation | Equipe produit / métier | Modification des règles de validation métier |
| Accès BDD | Equipe infrastructure | Changement de moteur SQL, migration vers un ORM |
| Rendu HTML | Equipe front-end / UX | Refonte du thème, migration vers un moteur de templates |
| Cache HTTP | Equipe infrastructure / DevOps | Stratégie de cache, passage à Redis ou un CDN |
| Journalisation | Equipe infrastructure | Changement d'outil de logs (fichier → ELK, Sentry...) |

Six acteurs → six responsabilités distinctes → six préoccupations à séparer.

### Étape 2 — Architecture en couches proposée

```
┌─────────────────────────────────────────────────────────┐
│  HTTP Layer                                              │
│  PostController  (routage + réponse HTTP + cache)        │
│  PostRequestValidator  (validation des données entrantes)│
└────────────────────────┬────────────────────────────────┘
                         │ appelle
┌────────────────────────▼────────────────────────────────┐
│  Application / Service Layer                             │
│  PostService  (orchestration des cas d'usage)            │
└────────────────────────┬────────────────────────────────┘
                         │ appelle
┌────────────────────────▼────────────────────────────────┐
│  Domain / Repository Layer                               │
│  PostRepositoryInterface  (contrat)                      │
│  SqlitePostRepository / PdoPostRepository  (concret)     │
└─────────────────────────────────────────────────────────┘

Transversal (injecté partout où nécessaire) :
  LoggerInterface / FileLogger / NullLogger
  PostView  (rendu HTML, indépendant du reste)
```

**Règle de dépendance** : les couches supérieures dépendent des couches inférieures via des interfaces, jamais via des classes concrètes.

### Étape 3 — Implémentation de `PostService` et `PostRepository`

```php
<?php

// ── Post.php (objet valeur / entité du domaine) ───────────────────────────

declare(strict_types=1);

final class Post
{
    public function __construct(
        public readonly ?int   $id,
        public readonly string $title,
        public readonly string $content,
        public readonly string $author,
        public readonly \DateTimeImmutable $createdAt,
    ) {}

    public static function create(string $title, string $content, string $author): self
    {
        return new self(
            id:        null,
            title:     $title,
            content:   $content,
            author:    $author,
            createdAt: new \DateTimeImmutable(),
        );
    }

    public function withId(int $id): self
    {
        return new self($id, $this->title, $this->content, $this->author, $this->createdAt);
    }
}


// ── PostRepositoryInterface.php ───────────────────────────────────────────

interface PostRepositoryInterface
{
    public function findById(int $id): ?Post;

    /** @return Post[] */
    public function findAll(): array;

    public function save(Post $post): Post;   // retourne le Post avec son id

    public function delete(int $id): void;
}


// ── PdoPostRepository.php (couche infrastructure) ────────────────────────

final class PdoPostRepository implements PostRepositoryInterface
{
    /**
     * Responsabilité unique : traduire les opérations métier Post en SQL
     * et les résultats SQL en objets Post.
     * Raison de changer : modification du schéma BDD, changement de moteur SQL.
     */
    public function __construct(private readonly PDO $pdo)
    {
        $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    }

    public function findById(int $id): ?Post
    {
        $stmt = $this->pdo->prepare(
            'SELECT id, title, content, author, created_at FROM posts WHERE id = :id'
        );
        $stmt->execute([':id' => $id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        return $row ? $this->hydrate($row) : null;
    }

    public function findAll(): array
    {
        $stmt = $this->pdo->query(
            'SELECT id, title, content, author, created_at FROM posts ORDER BY created_at DESC'
        );
        return array_map($this->hydrate(...), $stmt->fetchAll(PDO::FETCH_ASSOC));
    }

    public function save(Post $post): Post
    {
        if ($post->id === null) {
            return $this->insert($post);
        }
        return $this->update($post);
    }

    public function delete(int $id): void
    {
        $stmt = $this->pdo->prepare('DELETE FROM posts WHERE id = :id');
        $stmt->execute([':id' => $id]);
    }

    // ── Méthodes privées ──────────────────────────────────────────────────

    private function insert(Post $post): Post
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO posts (title, content, author, created_at)
             VALUES (:title, :content, :author, :created_at)'
        );
        $stmt->execute([
            ':title'      => $post->title,
            ':content'    => $post->content,
            ':author'     => $post->author,
            ':created_at' => $post->createdAt->format('Y-m-d H:i:s'),
        ]);
        return $post->withId((int) $this->pdo->lastInsertId());
    }

    private function update(Post $post): Post
    {
        $stmt = $this->pdo->prepare(
            'UPDATE posts SET title = :title, content = :content, author = :author
             WHERE id = :id'
        );
        $stmt->execute([
            ':title'   => $post->title,
            ':content' => $post->content,
            ':author'  => $post->author,
            ':id'      => $post->id,
        ]);
        return $post;
    }

    private function hydrate(array $row): Post
    {
        return new Post(
            id:        (int) $row['id'],
            title:     $row['title'],
            content:   $row['content'],
            author:    $row['author'],
            createdAt: new \DateTimeImmutable($row['created_at']),
        );
    }
}


// ── LoggerInterface.php ───────────────────────────────────────────────────

interface LoggerInterface
{
    public function error(string $message, array $context = []): void;
    public function info(string $message, array $context = []): void;
}

final class FileLogger implements LoggerInterface
{
    public function __construct(private readonly string $path) {}

    public function error(string $message, array $context = []): void
    {
        $this->write('ERROR', $message, $context);
    }

    public function info(string $message, array $context = []): void
    {
        $this->write('INFO', $message, $context);
    }

    private function write(string $level, string $message, array $context): void
    {
        $line = sprintf(
            "[%s] %s: %s %s\n",
            date('Y-m-d H:i:s'),
            $level,
            $message,
            $context ? json_encode($context) : ''
        );
        file_put_contents($this->path, $line, FILE_APPEND | LOCK_EX);
    }
}

/** Doublure de test : absorbe les logs sans rien écrire. */
final class NullLogger implements LoggerInterface
{
    public function error(string $message, array $context = []): void {}
    public function info(string $message, array $context = []): void {}
}


// ── PostService.php (couche service applicatif) ───────────────────────────

final class PostService
{
    /**
     * Responsabilité unique : orchestrer les cas d'usage du blog
     * (lire, créer, modifier, supprimer un article).
     * Raison de changer : modification des règles métier ou du flux d'un cas d'usage.
     *
     * Ne sait rien de SQL, ni de HTTP, ni de HTML.
     */
    public function __construct(
        private readonly PostRepositoryInterface $repository,
        private readonly LoggerInterface         $logger,
    ) {}

    public function getAll(): array
    {
        $this->logger->info('Récupération de tous les articles.');
        return $this->repository->findAll();
    }

    public function getById(int $id): Post
    {
        $post = $this->repository->findById($id);
        if ($post === null) {
            throw new \RuntimeException("Article introuvable : #$id");
        }
        return $post;
    }

    public function create(string $title, string $content, string $author): Post
    {
        $post = Post::create($title, $content, $author);
        $saved = $this->repository->save($post);
        $this->logger->info("Article créé.", ['id' => $saved->id]);
        return $saved;
    }

    public function update(int $id, string $title, string $content): Post
    {
        $existing = $this->getById($id);
        $updated  = new Post(
            id:        $existing->id,
            title:     $title,
            content:   $content,
            author:    $existing->author,
            createdAt: $existing->createdAt,
        );
        return $this->repository->save($updated);
    }

    public function delete(int $id): void
    {
        $this->getById($id); // lève une exception si l'article n'existe pas
        $this->repository->delete($id);
        $this->logger->info("Article supprimé.", ['id' => $id]);
    }
}
```

### Test unitaire de `PostService`

Sans base de données réelle : on injecte un faux dépôt en mémoire et un `NullLogger`.

```php
<?php

// ── InMemoryPostRepository.php (doublure de test) ─────────────────────────

final class InMemoryPostRepository implements PostRepositoryInterface
{
    private array $store = [];
    private int   $nextId = 1;

    public function findById(int $id): ?Post
    {
        return $this->store[$id] ?? null;
    }

    public function findAll(): array
    {
        return array_values($this->store);
    }

    public function save(Post $post): Post
    {
        if ($post->id === null) {
            $post = $post->withId($this->nextId++);
        }
        $this->store[$post->id] = $post;
        return $post;
    }

    public function delete(int $id): void
    {
        unset($this->store[$id]);
    }
}


// ── PostServiceTest.php (PHPUnit) ─────────────────────────────────────────

use PHPUnit\Framework\TestCase;

final class PostServiceTest extends TestCase
{
    private PostService $service;
    private InMemoryPostRepository $repository;

    protected function setUp(): void
    {
        $this->repository = new InMemoryPostRepository();
        $this->service    = new PostService($this->repository, new NullLogger());
    }

    public function testCreateStoresPost(): void
    {
        $post = $this->service->create('Mon titre', 'Contenu du post', 'Alice');

        $this->assertNotNull($post->id);
        $this->assertSame('Mon titre', $post->title);
        $this->assertSame('Alice', $post->author);
    }

    public function testGetByIdReturnsCorrectPost(): void
    {
        $created = $this->service->create('Test', 'Corps', 'Bob');
        $found   = $this->service->getById($created->id);

        $this->assertSame($created->id, $found->id);
    }

    public function testGetByIdThrowsWhenNotFound(): void
    {
        $this->expectException(\RuntimeException::class);
        $this->service->getById(999);
    }

    public function testDeleteRemovesPost(): void
    {
        $post = $this->service->create('A supprimer', 'Corps', 'Carol');
        $this->service->delete($post->id);

        $this->expectException(\RuntimeException::class);
        $this->service->getById($post->id);
    }

    public function testGetAllReturnsAllPosts(): void
    {
        $this->service->create('Post 1', 'Corps', 'Alice');
        $this->service->create('Post 2', 'Corps', 'Bob');

        $this->assertCount(2, $this->service->getAll());
    }

    public function testUpdateChangesContent(): void
    {
        $post    = $this->service->create('Ancien titre', 'Ancien corps', 'Alice');
        $updated = $this->service->update($post->id, 'Nouveau titre', 'Nouveau corps');

        $this->assertSame('Nouveau titre', $updated->title);
        $this->assertSame('Alice', $updated->author); // l'auteur est préservé
    }
}
```

### Étape 4 — Remplacement de SQLite par PostgreSQL sans toucher à `PostService`

Le point central est que `PostService` dépend de `PostRepositoryInterface`, jamais de `PdoPostRepository`. La migration se fait en trois étapes, sans modifier une seule ligne du service :

**1. Créer une nouvelle implémentation concrète**

```php
<?php

final class PostgresPostRepository implements PostRepositoryInterface
{
    // Même interface, même contrat, requêtes SQL adaptées à PostgreSQL.
    // Les colonnes SERIAL remplacent AUTOINCREMENT,
    // le placeholder $1 remplace ? dans les prepared statements, etc.

    public function __construct(private readonly PDO $pdo) {}

    public function findById(int $id): ?Post
    {
        $stmt = $this->pdo->prepare(
            'SELECT id, title, content, author, created_at FROM posts WHERE id = $1'
        );
        // ... même logique d'hydratation que PdoPostRepository
    }

    // ... findAll(), save(), delete() adaptés à PostgreSQL
}
```

**2. Reconfigurer la composition à la racine de l'application**

```php
<?php

// Avant (SQLite) :
$pdo        = new PDO('sqlite:/var/data/blog.db');
$repository = new PdoPostRepository($pdo);

// Après (PostgreSQL) : une seule ligne change.
$pdo        = new PDO('pgsql:host=db;dbname=blog', 'user', 'pass');
$repository = new PostgresPostRepository($pdo);

// Le service ne change pas :
$logger  = new FileLogger('/var/log/blog.log');
$service = new PostService($repository, $logger);
```

**3. Aucune modification du service ni des tests**

Les tests de `PostService` continuent d'utiliser `InMemoryPostRepository`. On peut écrire séparément des **tests d'intégration** pour `PostgresPostRepository` (avec une vraie connexion PostgreSQL de test) sans toucher au reste.

```
Couche testée              Doublure utilisée
─────────────────────────────────────────────
PostService                InMemoryPostRepository + NullLogger
PdoPostRepository          Base SQLite en mémoire (:memory:)
PostgresPostRepository     Base PostgreSQL de test (Docker)
PostController             Mock de PostService
```

### Récapitulatif des responsabilités finales

| Classe | Responsabilité unique | Raison de changer |
|---|---|---|
| `Post` | Représenter un article du domaine | Modification du modèle métier |
| `PostRepositoryInterface` | Définir le contrat d'accès aux données | Modification des besoins d'accès |
| `PdoPostRepository` | Persister les articles en SQL (PDO) | Changement de schéma ou de moteur SQL |
| `PostgresPostRepository` | Variante PostgreSQL du repository | Spécificités du dialecte PostgreSQL |
| `InMemoryPostRepository` | Doublure de test | Besoin des tests unitaires |
| `PostService` | Orchestrer les cas d'usage du blog | Modification des règles métier ou du flux |
| `LoggerInterface` | Définir le contrat de journalisation | Modification des besoins de log |
| `FileLogger` | Ecrire les logs dans un fichier texte | Changement de destination ou de format |
| `NullLogger` | Absorber les logs silencieusement (tests) | Besoins des tests unitaires |

---

## Synthèse des deux exercices

Ces deux exercices illustrent une progression naturelle :

- **L'exercice 2** montre que le SRP s'applique dès la conception initiale : identifier les acteurs *avant* d'écrire le code évite de créer des classes trop larges.
- **L'exercice 3** montre que le SRP s'applique aussi en refactorisation de code legacy : on découpe en partant des responsabilités observées, on crée des interfaces pour rendre le tout testable, et on valide la bonne séparation en vérifiant que chaque changement ne touche qu'une seule classe.

Dans les deux cas, les bénéfices concrets sont identiques :
- **Testabilité** : chaque classe peut être testée isolément avec des doublures simples.
- **Extensibilité** : ajouter un nouveau canal de notification (SMS, webhook) ou un nouveau moteur de BDD ne touche pas à la logique métier.
- **Lisibilité** : une classe courte avec une seule responsabilité est plus facile à comprendre et à faire évoluer.
