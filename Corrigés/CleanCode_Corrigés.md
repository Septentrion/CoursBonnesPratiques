
#### Corrigé de l'exercice A

| Smell identifié | Localisation | Refactorisation |
|---|---|---|
| Nommage opaque | Classe `Mgr`, méthodes `do`/`rep`, tous les attributs | Rename (classe, méthodes, variables) |
| Responsabilités multiples (SRP) | `do` : validation + BDD + calcul remise + email + historique | Extract Method puis Move Method vers des classes dédiées |
| Primitive Obsession | `t, d, e, p, q` sont des primitifs sans type ni contrainte | Introduce Parameter Object (`OrderRequest`) |
| Magic numbers | `0.05`, `0.10`, `0.15`, `10`, `50`, `100` | Extract Constant ou Extract Class (`DiscountPolicy`) |
| Feature Envy (calcul remise) | Le calcul de remise par quantité concerne la politique commerciale | Extract Class (`QuantityDiscountPolicy`) |
| Divergent Change | `do` changera si la BDD change, si l'email change, si la remise change | Séparer en `OrderRepository`, `OrderMailer`, `DiscountCalculator` |
| Imports dans le corps | `import sqlite3` et `import smtplib` dans `__init__` et `do` | Injection de dépendances |
| Gestion d'erreur silencieuse | `except Exception: pass` sur l'envoi e-mail | Journaliser l'erreur ou propager une exception métier |

**Proposition de découpage :**

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class OrderRequest:
    title:       str
    description: str
    email:       str
    product_id:  int
    quantity:    int


class QuantityDiscountPolicy:
    _TIERS = [(100, Decimal("0.15")), (50, Decimal("0.10")), (10, Decimal("0.05"))]

    def discount_rate(self, quantity: int) -> Decimal:
        for threshold, rate in self._TIERS:
            if quantity >= threshold:
                return rate
        return Decimal("0")

    def apply(self, amount: Decimal, quantity: int) -> Decimal:
        return amount * (1 - self.discount_rate(quantity))


class OrderValidator:
    def validate(self, request: OrderRequest) -> None:
        if not request.title.strip():
            raise ValueError("Le titre est obligatoire.")
        if "@" not in request.email or "." not in request.email:
            raise ValueError("Email invalide.")
        if request.quantity <= 0:
            raise ValueError("La quantité doit être positive.")


class OrderRepository:
    def __init__(self, conn):
        self._conn = conn

    def find_product(self, product_id: int) -> dict | None:
        row = self._conn.execute(
            "SELECT id, price, stock FROM products WHERE id = ?", (product_id,)
        ).fetchone()
        return dict(row) if row else None

    def create_order(self, request: OrderRequest, total: Decimal) -> None:
        self._conn.execute(
            "INSERT INTO orders (title, description, email, product_id, quantity, total) "
            "VALUES (?, ?, ?, ?, ?, ?)",
            (request.title, request.description, request.email,
             request.product_id, request.quantity, float(total))
        )
        self._conn.execute(
            "UPDATE products SET stock = stock - ? WHERE id = ?",
            (request.quantity, request.product_id)
        )
        self._conn.commit()


class OrderMailer:
    def __init__(self, smtp_host: str = "localhost"):
        self._host = smtp_host

    def send_confirmation(self, email: str, quantity: int, total: Decimal) -> None:
        import smtplib
        from email.message import EmailMessage
        msg = EmailMessage()
        msg["To"]      = email
        msg["From"]    = "shop@example.com"
        msg["Subject"] = "Commande confirmée"
        msg.set_content(f"Votre commande de {quantity} unité(s) pour {total:.2f} € est confirmée.")
        with smtplib.SMTP(self._host) as smtp:
            smtp.send_message(msg)


class OrderService:
    def __init__(
        self,
        validator:   OrderValidator,
        repository:  OrderRepository,
        discount:    QuantityDiscountPolicy,
        mailer:      OrderMailer,
    ):
        self._validator  = validator
        self._repository = repository
        self._discount   = discount
        self._mailer     = mailer

    def place_order(self, request: OrderRequest) -> Decimal:
        self._validator.validate(request)

        product = self._repository.find_product(request.product_id)
        if product is None:
            raise ValueError(f"Produit introuvable : {request.product_id}")
        if request.quantity > product["stock"]:
            raise ValueError("Stock insuffisant.")

        subtotal = Decimal(str(product["price"])) * request.quantity
        total    = self._discount.apply(subtotal, request.quantity)

        self._repository.create_order(request, total)
        self._mailer.send_confirmation(request.email, request.quantity, total)

        return total
```

---

#### Corrigé de l'exercice B

Les responsabilités identifiées dans `Cart` :
1. Gestion des articles (collection d'items, fusion des quantités, comptage, vidage)
2. Application des codes promo (table des remises, calcul du total)
3. Rendu HTML (présentation)

Découpage proposé :

```php
<?php

declare(strict_types=1);

// ── CartItem.php — objet valeur immuable ──────────────────────────────────────

final class CartItem
{
    public function __construct(
        public readonly int    $productId,
        public readonly string $name,
        public readonly float  $unitPrice,
        public readonly int    $quantity,
    ) {}

    public function withQuantity(int $quantity): self
    {
        return new self($this->productId, $this->name, $this->unitPrice, $quantity);
    }

    public function lineTotal(): float
    {
        return $this->unitPrice * $this->quantity;
    }
}


// ── PromoCode.php — objet valeur ──────────────────────────────────────────────

final class PromoCode
{
    private const VALID_CODES = [
        'SUMMER' => 10,
        'VIP'    => 25,
        'STAFF'  => 50,
    ];

    private function __construct(
        public readonly string $code,
        public readonly int    $discountPercent,
    ) {}

    public static function fromString(string $code): ?self
    {
        if (!array_key_exists($code, self::VALID_CODES)) {
            return null;
        }
        return new self($code, self::VALID_CODES[$code]);
    }

    public function applyTo(float $amount): float
    {
        return $amount * (1 - $this->discountPercent / 100);
    }
}


// ── Cart.php — gestion des articles et calcul du total ───────────────────────

final class Cart
{
    /** @var CartItem[] */
    private array      $items = [];
    private ?PromoCode $promoCode = null;

    public function add(int $productId, string $name, float $price, int $quantity): void
    {
        foreach ($this->items as $index => $existing) {
            if ($existing->productId === $productId) {
                $this->items[$index] = $existing->withQuantity(
                    $existing->quantity + $quantity
                );
                return;
            }
        }
        $this->items[] = new CartItem($productId, $name, $price, $quantity);
    }

    public function setCode(string $code): void
    {
        $this->promoCode = PromoCode::fromString($code);
        // Si le code est invalide, PromoCode::fromString retourne null —
        // le panier reste sans code promo, sans exception.
    }

    public function calc(): float
    {
        $subtotal = array_sum(
            array_map(fn(CartItem $item) => $item->lineTotal(), $this->items)
        );
        return $this->promoCode !== null
            ? $this->promoCode->applyTo($subtotal)
            : $subtotal;
    }

    public function count(): int
    {
        return count($this->items);
    }

    public function clear(): void
    {
        $this->items     = [];
        $this->promoCode = null;
    }

    /** @return CartItem[] */
    public function items(): array
    {
        return $this->items;
    }

    public function promoCode(): ?PromoCode
    {
        return $this->promoCode;
    }
}


// ── CartHtmlRenderer.php — rendu HTML séparé ─────────────────────────────────

final class CartHtmlRenderer
{
    public function render(Cart $cart): string
    {
        $rows = implode('', array_map(
            fn(CartItem $item) => sprintf(
                '<tr><td>%s</td><td>%d</td><td>%s €</td><td>%s €</td></tr>',
                htmlspecialchars($item->name),
                $item->quantity,
                number_format($item->unitPrice, 2),
                number_format($item->lineTotal(), 2),
            ),
            $cart->items()
        ));

        $total  = '<p>Total : ' . number_format($cart->calc(), 2) . ' €</p>';
        $promo  = '';
        $code   = $cart->promoCode();
        if ($code !== null) {
            $promo = sprintf(
                '<p>Code promo appliqué : %s (-%d%%)</p>',
                htmlspecialchars($code->code),
                $code->discountPercent,
            );
        }

        return "<table>{$rows}</table>{$total}{$promo}";
    }
}
```

Les tests originaux passent sans modification car les signatures de `add()`,
`setCode()`, `calc()`, `count()` et `clear()` sont préservées. `html()` a été
extraite dans `CartHtmlRenderer::render()` — si un test existant l'appelait, il
faudrait adapter, mais aucun des tests fournis ne le fait.

---

#### Corrigé de l'exercice C — Mikado Map

**Graphe de la Mikado Map :**

```
[Objectif] UserController testable sans base de données
    │
    ├── [A] UserController dépend de UserRepositoryInterface (DIP)
    │       ├── [A1] Extraire l'interface UserRepositoryInterface
    │       │         (méthodes : findById, findAll)
    │       └── [A2] Injecter UserRepositoryInterface dans le constructeur
    │                 (supprimer les new PDO() dans les méthodes)
    │
    ├── [B] PdoUserRepository implémente UserRepositoryInterface
    │       └── [B1] Créer PdoUserRepository avec PDO injecté
    │
    ├── [C] InMemoryUserRepository implémente UserRepositoryInterface
    │       (pour les tests — aucune dépendance à PDO)
    │
    └── [D] UserController ne gère plus les codes HTTP directement
            (extraction de la logique de réponse dans un ResponseEmitter)
            └── [D1] Optionnel : séparer la logique de vue du contrôleur
```

**Ordre d'exécution (feuilles en premier) :**

1. `A1` — Extraire `UserRepositoryInterface` (deux méthodes, pas de dépendance)
2. `B1` — Créer `PdoUserRepository implements UserRepositoryInterface`
3. `C`  — Créer `InMemoryUserRepository implements UserRepositoryInterface`
4. `A2` — Modifier `UserController` pour recevoir `UserRepositoryInterface` en paramètre
5. `A`  — Vérifier que les tests passent avec `InMemoryUserRepository`
6. `D1` (optionnel) — Extraire le rendu HTML dans une classe `UserView`
7. Objectif atteint

**Code résultant (étapes 1 à 5) :**

```php
<?php

interface UserRepositoryInterface
{
    public function findById(int $id): ?array;
    /** @return array[] */
    public function findAll(): array;
}

final class PdoUserRepository implements UserRepositoryInterface
{
    public function __construct(private readonly PDO $pdo) {}

    public function findById(int $id): ?array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$id]);
        return $stmt->fetch(PDO::FETCH_ASSOC) ?: null;
    }

    public function findAll(): array
    {
        return $this->pdo->query('SELECT id, name FROM users ORDER BY name')
                         ->fetchAll(PDO::FETCH_ASSOC);
    }
}

final class InMemoryUserRepository implements UserRepositoryInterface
{
    private array $users;

    public function __construct(array $users = [])
    {
        // Indexé par id pour la recherche en O(1)
        $this->users = array_column($users, null, 'id');
    }

    public function findById(int $id): ?array
    {
        return $this->users[$id] ?? null;
    }

    public function findAll(): array
    {
        $all = array_values($this->users);
        usort($all, fn($a, $b) => strcmp($a['name'], $b['name']));
        return $all;
    }
}

final class UserController
{
    public function __construct(
        private readonly UserRepositoryInterface $repository
    ) {}

    public function show(int $id): string
    {
        $user = $this->repository->findById($id);
        if ($user === null) {
            http_response_code(404);
            return '<h1>Utilisateur introuvable</h1>';
        }
        return '<h1>' . htmlspecialchars($user['name']) . '</h1>';
    }

    public function index(): string
    {
        $rows = $this->repository->findAll();
        $items = implode('', array_map(
            fn($row) => '<li>' . htmlspecialchars($row['name']) . '</li>',
            $rows
        ));
        return "<ul>{$items}</ul>";
    }
}

// Test unitaire — aucune BDD nécessaire
final class UserControllerTest extends PHPUnit\Framework\TestCase
{
    public function testShowReturnsUserName(): void
    {
        $repo       = new InMemoryUserRepository([['id' => 1, 'name' => 'Alice']]);
        $controller = new UserController($repo);
        $this->assertStringContainsString('Alice', $controller->show(1));
    }

    public function testShowReturns404ForMissingUser(): void
    {
        $controller = new UserController(new InMemoryUserRepository());
        $html = $controller->show(999);
        $this->assertStringContainsString('introuvable', $html);
    }
}
```
