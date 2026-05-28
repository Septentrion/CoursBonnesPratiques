# Jour 2 — Matin : Programmation fonctionnelle & TDD

**Durée totale :** 3 h 30 (deux sessions séparées par une pause de 15 minutes)  
**Public :** Etudiants en informatique, connaissance préalable des design patterns,
des principes SOLID, du Clean Code (jour 1 matin) et du DDD (jour 1 après-midi)  
**Langages illustrés :** PHP 8.x et Python 3.10+

---

## Avant-propos : deux paradigmes, un seul objectif

La journée précédente a posé les bases d'un code orienté objet expressif et bien
structuré. L'orienté objet excelle pour modéliser des entités ayant un état, un
cycle de vie et des comportements — exactement ce que le DDD exploite.

Mais l'orienté objet porte un coût structurel : les objets ont un état mutable,
ce qui rend le raisonnement sur le code plus difficile. Qu'est-ce que cet objet
contient à ce moment précis du programme ? Est-ce que cette méthode produit
toujours le même résultat si je l'appelle deux fois de suite ?

La programmation fonctionnelle répond à ces questions en imposant une contrainte
radicale : **les fonctions ne modifient pas l'état, elles transforment des valeurs**.
Ce n'est pas un paradigme concurrent de l'orienté objet — c'est un outil
complémentaire. Les meilleurs programmeurs savent quand utiliser l'un, l'autre,
ou les deux ensemble.

---

## Session 5 — Programmation fonctionnelle dans un monde objet 

### 5.1 Immutabilité et fonctions pures

#### Fonctions pures

Une fonction pure satisfait deux propriétés :

1. **Déterminisme** : pour les mêmes arguments, elle retourne toujours le même
   résultat.
2. **Absence d'effets de bord** : elle ne modifie aucun état extérieur (pas
   d'écriture en BDD, pas d'appel réseau, pas de modification de variables
   globales, pas de mutation de ses arguments).

```python
# Impure : résultat non déterministe (dépend de l'heure)
def get_greeting(name: str) -> str:
    from datetime import datetime
    hour = datetime.now().hour
    prefix = "Bonjour" if hour < 18 else "Bonsoir"
    return f"{prefix}, {name} !"

# Impure : effet de bord (mutation d'une variable externe)
total = 0
def add_to_total(amount: int) -> None:
    global total
    total += amount   # modifie l'état global

# Pure : même entrée → même sortie, aucun effet de bord
def add(a: int, b: int) -> int:
    return a + b

# Pure : ne modifie pas la liste d'entrée, retourne une nouvelle liste
def append_item(items: list, item) -> list:
    return [*items, item]
```

```php
<?php
// Impure : dépend d'une variable globale externe
function applyDiscount(float $price): float {
    global $discountRate;         // dépendance cachée
    return $price * (1 - $discountRate);
}

// Pure : toutes les dépendances sont des paramètres explicites
function applyDiscount(float $price, float $discountRate): float {
    return $price * (1 - $discountRate);
}
```

**Pourquoi les fonctions pures facilitent-elles le raisonnement ?**

On peut remplacer mentalement un appel de fonction pure par sa valeur de retour
(*referential transparency*). `add(2, 3)` est toujours `5` — partout, tout le
temps. Un test est trivial : pas de setup, pas de teardown, pas de mock.

#### Immutabilité

Un objet immuable ne peut pas être modifié après sa création. Toute « modification »
produit un nouvel objet avec les nouvelles valeurs.

```python
# Mutable — difficile à raisonner : qui a modifié cet objet et quand ?
class ShoppingCart:
    def __init__(self):
        self.items = []
        self.promo_code = None

    def add_item(self, item):
        self.items.append(item)    # mutation en place

    def apply_promo(self, code):
        self.promo_code = code     # mutation en place


# Immuable — chaque opération retourne un nouvel état
from dataclasses import dataclass, field, replace
from typing import Optional

@dataclass(frozen=True)
class CartItem:
    product_id: str
    quantity:   int
    unit_price: float

    def total(self) -> float:
        return self.quantity * self.unit_price

@dataclass(frozen=True)
class ShoppingCart:
    items:      tuple[CartItem, ...]  = field(default_factory=tuple)
    promo_code: Optional[str]         = None

    def add_item(self, item: CartItem) -> "ShoppingCart":
        return replace(self, items=(*self.items, item))

    def apply_promo(self, code: str) -> "ShoppingCart":
        return replace(self, promo_code=code)

    def subtotal(self) -> float:
        return sum(i.total() for i in self.items)

# Usage : chaque opération crée un nouveau cart, l'original est inchangé
empty_cart  = ShoppingCart()
cart_with_item = empty_cart.add_item(CartItem("P001", 2, 19.99))
cart_with_promo = cart_with_item.apply_promo("SUMMER10")

assert empty_cart.items == ()       # inchangé
assert len(cart_with_item.items) == 1
```

```php
<?php
// Immuable en PHP : le Value Object vu hier est déjà immuable
final class Money
{
    public function __construct(
        public readonly int    $amountCents,
        public readonly string $currency,
    ) {}

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException("Devises incompatibles.");
        }
        return new self($this->amountCents + $other->amountCents, $this->currency);
    }

    public function multiply(float $factor): self
    {
        return new self((int) round($this->amountCents * $factor), $this->currency);
    }
}
```

**Performance et immutabilité :** l'objection classique est que créer un nouvel
objet à chaque opération est coûteux. En pratique, les langages modernes et leurs
GC gèrent efficacement les objets de courte durée de vie. Pour les structures
larges (tableaux de millions d'éléments), des structures persistantes (*persistent
data structures*) comme celles de Clojure ou de la bibliothèque `pyrsistent` en
Python partagent la mémoire structurellement et rendent les copies peu coûteuses.

---

### 5.2 Higher-order functions, closures, currying et composition

#### Fonctions d'ordre supérieur (HOF)

Une HOF est une fonction qui prend d'autres fonctions comme arguments, ou qui
en retourne une. Elles constituent le mécanisme de base pour abstraire des
comportements et éviter la duplication.

```python
# map, filter, reduce sont les HOF canoniques
from functools import reduce

prices     = [10.0, 25.5, 8.0, 42.0, 3.5]
discounted = list(map(lambda p: p * 0.9, prices))
expensive  = list(filter(lambda p: p > 10, prices))
total      = reduce(lambda acc, p: acc + p, prices, 0.0)

# Equivalent avec compréhensions (plus pythonique) :
discounted = [p * 0.9 for p in prices]
expensive  = [p for p in prices if p > 10]
total      = sum(prices)

# HOF custom : appliquer une transformation conditionnelle
def transform_if(predicate, transform, items):
    return [transform(x) if predicate(x) else x for x in items]

# Appliquer une remise de 20% uniquement sur les articles > 20 €
result = transform_if(lambda p: p > 20, lambda p: p * 0.8, prices)
```

```php
<?php
// array_map, array_filter, array_reduce sont les HOF PHP équivalentes

$prices     = [10.0, 25.5, 8.0, 42.0, 3.5];
$discounted = array_map(fn(float $p) => $p * 0.9, $prices);
$expensive  = array_filter($prices, fn(float $p) => $p > 10);
$total      = array_reduce($prices, fn(float $carry, float $p) => $carry + $p, 0.0);

// HOF custom en PHP
function transformIf(callable $predicate, callable $transform, array $items): array
{
    return array_map(
        fn($x) => $predicate($x) ? $transform($x) : $x,
        $items
    );
}

$result = transformIf(
    fn(float $p) => $p > 20,
    fn(float $p) => $p * 0.8,
    $prices,
);
```

#### Closures

Une closure est une fonction qui « capture » des variables de son environnement
lexical. Elle transporte avec elle les données dont elle a besoin.

```python
# La closure capture le paramètre `rate` de la fonction englobante
def make_discount_function(rate: float):
    def apply(price: float) -> float:
        return price * (1 - rate)   # `rate` est capturé
    return apply

discount_10 = make_discount_function(0.10)
discount_25 = make_discount_function(0.25)

print(discount_10(100))   # 90.0
print(discount_25(100))   # 75.0
```

```php
<?php
// Même pattern en PHP avec use()
function makeDiscountFunction(float $rate): \Closure
{
    return function(float $price) use ($rate): float {
        return $price * (1 - $rate);
    };
}

$discount10 = makeDiscountFunction(0.10);
$discount25 = makeDiscountFunction(0.25);
```

#### Currying et application partielle

Le **currying** transforme une fonction à n paramètres en une chaîne de n fonctions
à un paramètre. L'**application partielle** fixe certains arguments d'une fonction
pour en créer une nouvelle.

```python
from functools import partial

# Application partielle avec partial()
def calculate_tax(rate: float, amount: float) -> float:
    return amount * rate

french_vat    = partial(calculate_tax, 0.20)
reduced_vat   = partial(calculate_tax, 0.055)

print(french_vat(100))    # 20.0
print(reduced_vat(100))   # 5.5

# Currying manuel
def add(a: int):
    def inner(b: int) -> int:
        return a + b
    return inner

add5 = add(5)
print(add5(3))    # 8
print(add5(10))   # 15
```

```php
<?php
// Application partielle en PHP
function multiply(float $factor, float $value): float
{
    return $factor * $value;
}

$double = fn(float $x) => multiply(2, $x);
$triple = fn(float $x) => multiply(3, $x);

$prices  = [10.0, 20.0, 30.0];
$doubled = array_map($double, $prices);  // [20.0, 40.0, 60.0]
```

#### Composition de fonctions

La composition enchaîne des fonctions de sorte que la sortie de l'une devienne
l'entrée de la suivante. Elle est l'équivalent fonctionnel du chaînage de
méthodes en OO.

```python
from typing import Callable, TypeVar

T = TypeVar("T")

def compose(*functions: Callable) -> Callable:
    """
    compose(f, g, h)(x) == f(g(h(x)))
    Les fonctions sont appliquées de droite à gauche.
    """
    from functools import reduce
    return reduce(lambda f, g: lambda x: f(g(x)), functions)

def pipe(*functions: Callable) -> Callable:
    """
    pipe(f, g, h)(x) == h(g(f(x)))
    Les fonctions sont appliquées de gauche à droite (plus naturel à lire).
    """
    from functools import reduce
    return reduce(lambda f, g: lambda x: g(f(x)), functions)


# Pipeline de traitement d'une chaîne de caractères
normalize   = str.lower
strip_spaces = str.strip
remove_accents = lambda s: s.replace("é", "e").replace("à", "a")  # simplifié

clean_string = pipe(strip_spaces, normalize, remove_accents)
print(clean_string("  Bonjour à Tous  "))   # "bonjour a tous"


# Pipeline de traitement d'un montant
apply_vat        = lambda amount: amount * 1.20
apply_discount   = lambda amount: amount * 0.90
round_to_cents   = lambda amount: round(amount, 2)

compute_final_price = pipe(apply_discount, apply_vat, round_to_cents)
print(compute_final_price(100.0))   # 108.0
```

---

### 5.3 Gestion des effets de bord : Maybe et Either

En programmation fonctionnelle, les valeurs qui peuvent être absentes ou les
opérations qui peuvent échouer sont représentées explicitement dans les types.
Cela évite les `None` silencieux et les exceptions incontrôlées.

#### Maybe / Option — valeur potentiellement absente

`Maybe` (ou `Option`) est un type qui contient soit *une valeur* (`Just` / `Some`),
soit *rien* (`Nothing` / `None`). Il force le code appelant à gérer l'absence
de valeur explicitement — impossible d'oublier le `if result is not None`.

```python
from __future__ import annotations
from typing import TypeVar, Generic, Callable, Optional

T = TypeVar("T")
U = TypeVar("U")


class Maybe(Generic[T]):
    """
    Type Maybe/Option : encapsule une valeur potentiellement absente.
    Aucune exception ne s'échappe de cette classe.
    """

    def __init__(self, value: Optional[T]):
        self._value = value

    @classmethod
    def just(cls, value: T) -> "Maybe[T]":
        if value is None:
            raise ValueError("Just ne peut pas contenir None. Utilisez Maybe.nothing().")
        return cls(value)

    @classmethod
    def nothing(cls) -> "Maybe[T]":
        return cls(None)

    @classmethod
    def of(cls, value: Optional[T]) -> "Maybe[T]":
        return cls(value)

    def is_present(self) -> bool:
        return self._value is not None

    def get_or_else(self, default: T) -> T:
        return self._value if self._value is not None else default

    def map(self, func: Callable[[T], U]) -> "Maybe[U]":
        """Applique func si la valeur est présente, retourne Nothing sinon."""
        if self._value is None:
            return Maybe.nothing()
        return Maybe.of(func(self._value))

    def flat_map(self, func: Callable[[T], "Maybe[U]"]) -> "Maybe[U]":
        """Comme map, mais func retourne elle-même un Maybe."""
        if self._value is None:
            return Maybe.nothing()
        return func(self._value)

    def __repr__(self) -> str:
        return f"Just({self._value!r})" if self._value is not None else "Nothing"


# Usage : chaîne de transformations sans vérification manuelle de None
users_db = {
    1: {"name": "Alice", "address_id": 42},
    2: {"name": "Bob",   "address_id": None},
}
addresses_db = {
    42: {"city": "Paris", "zip": "75001"},
}

def find_user(user_id: int) -> Maybe[dict]:
    return Maybe.of(users_db.get(user_id))

def find_address(address_id: int) -> Maybe[dict]:
    return Maybe.of(addresses_db.get(address_id))

def get_city_for_user(user_id: int) -> str:
    return (
        find_user(user_id)
        .flat_map(lambda u: Maybe.of(u.get("address_id")))
        .flat_map(find_address)
        .map(lambda a: a["city"])
        .get_or_else("Ville inconnue")
    )

print(get_city_for_user(1))   # "Paris"
print(get_city_for_user(2))   # "Ville inconnue" (address_id est None)
print(get_city_for_user(99))  # "Ville inconnue" (utilisateur inexistant)
```

#### Either / Result — opération qui peut échouer

`Either` contient soit un résultat de *succès* (`Right`), soit une valeur
d'*échec* (`Left`). Contrairement aux exceptions, l'échec est explicite dans
le type de retour — le code appelant *doit* le traiter.

```python
from __future__ import annotations
from typing import TypeVar, Generic, Callable, Union

L = TypeVar("L")  # Left  — type de l'erreur
R = TypeVar("R")  # Right — type du succès
S = TypeVar("S")


class Either(Generic[L, R]):
    """
    Either[L, R] : succès (Right) ou échec (Left).
    Convention : Left = erreur, Right = succès ("right" = correct).
    """

    def __init__(self, value: Union[L, R], is_right: bool):
        self._value    = value
        self._is_right = is_right

    @classmethod
    def right(cls, value: R) -> "Either[L, R]":
        return cls(value, True)

    @classmethod
    def left(cls, error: L) -> "Either[L, R]":
        return cls(error, False)

    def is_success(self) -> bool: return self._is_right
    def is_failure(self) -> bool: return not self._is_right

    def map(self, func: Callable[[R], S]) -> "Either[L, S]":
        if self._is_right:
            return Either.right(func(self._value))
        return Either.left(self._value)

    def flat_map(self, func: Callable[[R], "Either[L, S]"]) -> "Either[L, S]":
        if self._is_right:
            return func(self._value)
        return Either.left(self._value)

    def get_or_raise(self) -> R:
        if self._is_right:
            return self._value
        raise RuntimeError(str(self._value))

    def fold(self, on_left: Callable[[L], S], on_right: Callable[[R], S]) -> S:
        return on_right(self._value) if self._is_right else on_left(self._value)

    def __repr__(self) -> str:
        kind = "Right" if self._is_right else "Left"
        return f"{kind}({self._value!r})"
```

#### Railway-Oriented Programming

Le Railway-Oriented Programming (ROP), popularisé par Scott Wlaschin, est une
métaphore visuelle pour enchaîner des opérations susceptibles d'échouer. Chaque
étape est comme un aiguillage ferroviaire : si elle réussit, le train continue
sur la voie principale (Right) ; si elle échoue, il bascule sur la voie d'erreur
(Left) et toutes les étapes suivantes sont court-circuitées.

```python
from decimal import Decimal
from dataclasses import dataclass


@dataclass
class OrderRequest:
    customer_id: str
    product_id:  str
    quantity:    int
    promo_code:  str = ""


@dataclass(frozen=True)
class ValidatedOrder:
    customer_id: str
    product_id:  str
    quantity:    int
    unit_price:  Decimal
    discount:    Decimal


# Chaque étape retourne un Either
def validate_quantity(request: OrderRequest) -> Either[str, OrderRequest]:
    if request.quantity <= 0:
        return Either.left("La quantité doit être strictement positive.")
    if request.quantity > 1000:
        return Either.left("La quantité ne peut pas dépasser 1000.")
    return Either.right(request)


def fetch_price(request: OrderRequest) -> Either[str, tuple[OrderRequest, Decimal]]:
    prices = {"P001": Decimal("29.99"), "P002": Decimal("149.00")}
    price  = prices.get(request.product_id)
    if price is None:
        return Either.left(f"Produit introuvable : {request.product_id}")
    return Either.right((request, price))


def apply_promo(
    data: tuple[OrderRequest, Decimal]
) -> Either[str, ValidatedOrder]:
    request, price = data
    promos = {"SUMMER10": Decimal("0.10"), "VIP25": Decimal("0.25")}
    discount = promos.get(request.promo_code, Decimal("0"))
    return Either.right(ValidatedOrder(
        customer_id=request.customer_id,
        product_id=request.product_id,
        quantity=request.quantity,
        unit_price=price,
        discount=discount,
    ))


def compute_total(order: ValidatedOrder) -> Either[str, Decimal]:
    subtotal = order.unit_price * order.quantity
    total    = subtotal * (1 - order.discount)
    if total < Decimal("0"):
        return Either.left("Le total calculé est négatif — vérifiez les paramètres.")
    return Either.right(total.quantize(Decimal("0.01")))


# Pipeline Railway : les erreurs court-circuitent automatiquement
def process_order(request: OrderRequest) -> Either[str, Decimal]:
    return (
        Either.right(request)
        .flat_map(validate_quantity)
        .flat_map(fetch_price)
        .flat_map(apply_promo)
        .flat_map(compute_total)
    )


# Cas de succès
result = process_order(OrderRequest("C001", "P001", 3, "SUMMER10"))
result.fold(
    on_left=lambda err: print(f"Erreur : {err}"),
    on_right=lambda total: print(f"Total : {total} €"),
)
# Total : 80.97 €

# Cas d'échec — court-circuitage immédiat à la première erreur
result = process_order(OrderRequest("C001", "P999", 3))
result.fold(
    on_left=lambda err: print(f"Erreur : {err}"),
    on_right=lambda total: print(f"Total : {total} €"),
)
# Erreur : Produit introuvable : P999
```

---

### 5.4 Hybridation OO + FP — où chaque paradigme excelle

Aucun paradigme n'est universellement supérieur. Le tableau suivant résume les
forces de chacun pour guider les choix de conception :

| Problème | OO (Orienté Objet) | FP (Fonctionnel) |
|---|---|---|
| Modéliser des entités avec état et cycle de vie | Excellent (Aggregate, Entity) | Difficile |
| Transformer des données en pipeline | Verbeux | Excellent (map/filter/reduce) |
| Gérer des règles métier complexes | Excellent (polymorphisme) | Acceptable |
| Gérer des erreurs explicitement dans les types | Acceptable | Excellent (Either/Maybe) |
| Tester unitairement | Nécessite des mocks si état partagé | Trivial pour les fonctions pures |
| Traitement concurrent / parallèle | Délicat (accès concurrent à l'état) | Naturel (immuabilité) |
| Modéliser des variations de comportement | Excellent (Strategy, Visitor) | Fonctions d'ordre supérieur |

**Pattern hybride recommandé :**

```
Couche domaine (DDD)      → OO : Aggregates, Entities, Value Objects
Couche application        → Hybride : Application Services orchestrent,
                            pipelines fonctionnels transforment les données
Couche infrastructure     → OO : adapters, repositories
Transformations de données → FP : map/filter/reduce, pipelines purs
Gestion des erreurs        → FP : Either/Result pour les erreurs métier attendues
                              Exceptions pour les erreurs inattendues (programmation)
```

```php
<?php
// Hybridation en PHP : OO pour la structure, FP pour les transformations

final class InvoiceReportService
{
    public function generateMonthlyReport(
        array $invoices,    // array d'Invoice (OO)
        int   $year,
        int   $month,
    ): array {
        return array_values(array_map(
            // Transformation fonctionnelle pure sur des objets OO
            fn(Invoice $inv) => [
                'id'       => (string) $inv->id(),
                'customer' => $inv->customerName(),
                'total'    => $inv->total()->format(),
                'status'   => $inv->status()->label(),
            ],
            array_filter(
                $invoices,
                fn(Invoice $inv) => $inv->isInMonth($year, $month)
                                 && $inv->isPaid(),
            ),
        ));
    }
}
```

---

## Session 6 — Atelier : TDD & tests avancés

### 6.1 TDD — Test-Driven Development

#### Le cycle Red / Green / Refactor

Le TDD est une discipline de développement introduite par Kent Beck dans
*Test-Driven Development: By Example* (2002). Elle repose sur un cycle court
de trois phases, répété aussi souvent que nécessaire :

```
RED    → Ecrire un test qui échoue (le comportement n'existe pas encore).
GREEN  → Ecrire le minimum de code pour faire passer le test.
REFACTOR → Améliorer le code sans changer son comportement (les tests restent verts).
```

La règle fondamentale : **ne jamais écrire du code de production sans un test
qui échoue d'abord**. Cette contrainte peut sembler artificielle — elle est en
réalité un outil de conception. En écrivant le test avant l'implémentation, on
réfléchit d'abord à l'interface (comment ce code sera utilisé) avant aux détails
d'implémentation.

**Bénéfices du TDD :**
- Le code est testable par construction (on a écrit les tests en premier).
- Les tests servent de documentation vivante des comportements attendus.
- Le refactoring est sûr : les tests signalent immédiatement toute régression.
- La conception émerge progressivement plutôt que d'être planifiée à l'avance.

---

### 6.2 Kata TDD guidé — FizzBuzz étendu

Nous allons appliquer le cycle TDD sur un kata progressif. Un kata est un
exercice répété pour développer des automatismes.

**Règles du FizzBuzz étendu :**

- Nombre divisible par 3 → `"Fizz"`
- Nombre divisible par 5 → `"Buzz"`
- Nombre divisible par 7 → `"Whizz"`
- Divisible par 3 et 5 → `"FizzBuzz"`
- Divisible par 3 et 7 → `"FizzWhizz"`
- Divisible par 5 et 7 → `"BuzzWhizz"`
- Divisible par 3, 5 et 7 → `"FizzBuzzWhizz"`
- Aucune règle → le nombre en chaîne

La règle métier *peut changer* : demain, on veut ajouter `"Jazz"` pour les
multiples de 11. L'implémentation doit être extensible (OCP).

#### Déroulement du cycle TDD, itération par itération

**Itération 1 — RED**

```python
# test_fizzbuzz.py

def test_returns_number_as_string_when_no_rule_applies():
    assert fizzbuzz(1) == "1"
```

Ce test échoue immédiatement : `fizzbuzz` n'existe pas.

**Itération 1 — GREEN** (minimum de code)

```python
# fizzbuzz.py
def fizzbuzz(n: int) -> str:
    return str(n)
```

**Itération 1 — REFACTOR** : rien à refactoriser.

---

**Itération 2 — RED**

```python
def test_multiples_of_3_return_fizz():
    assert fizzbuzz(3) == "Fizz"
    assert fizzbuzz(6) == "Fizz"
    assert fizzbuzz(9) == "Fizz"
```

**Itération 2 — GREEN**

```python
def fizzbuzz(n: int) -> str:
    if n % 3 == 0:
        return "Fizz"
    return str(n)
```

---

**Itération 3 à 5** : ajouter Buzz (5), Whizz (7), puis les combinaisons.

Après cinq itérations, l'implémentation naïve ressemblerait à :

```python
def fizzbuzz(n: int) -> str:
    result = ""
    if n % 3 == 0: result += "Fizz"
    if n % 5 == 0: result += "Buzz"
    if n % 7 == 0: result += "Whizz"
    return result if result else str(n)
```

**Itération 6 — REFACTOR** : rendre l'ajout de nouvelles règles extensible

```python
# Après refactoring : règles déclaratives, OCP respecté

from dataclasses import dataclass

@dataclass(frozen=True)
class FizzBuzzRule:
    divisor: int
    label:   str

    def applies_to(self, n: int) -> bool:
        return n % self.divisor == 0


DEFAULT_RULES = [
    FizzBuzzRule(3,  "Fizz"),
    FizzBuzzRule(5,  "Buzz"),
    FizzBuzzRule(7,  "Whizz"),
]


def fizzbuzz(n: int, rules: list[FizzBuzzRule] = DEFAULT_RULES) -> str:
    result = "".join(r.label for r in rules if r.applies_to(n))
    return result if result else str(n)


# Ajouter Jazz pour 11 : aucune modification du code existant
EXTENDED_RULES = [*DEFAULT_RULES, FizzBuzzRule(11, "Jazz")]

# Les tests existants continuent de passer avec DEFAULT_RULES
```

**Tests complets après refactoring :**

```python
import pytest


@pytest.mark.parametrize("n, expected", [
    (1,   "1"),
    (2,   "2"),
    (4,   "4"),
    (3,   "Fizz"),
    (6,   "Fizz"),
    (5,   "Buzz"),
    (10,  "Buzz"),
    (7,   "Whizz"),
    (14,  "Whizz"),
    (15,  "FizzBuzz"),
    (21,  "FizzWhizz"),
    (35,  "BuzzWhizz"),
    (105, "FizzBuzzWhizz"),
])
def test_fizzbuzz_default_rules(n: int, expected: str):
    assert fizzbuzz(n) == expected


def test_custom_rule_can_be_added_without_modifying_fizzbuzz():
    jazz_rules = [*DEFAULT_RULES, FizzBuzzRule(11, "Jazz")]
    assert fizzbuzz(11, jazz_rules) == "Jazz"
    assert fizzbuzz(33, jazz_rules) == "FizzJazz"
    # Les règles de base restent inchangées
    assert fizzbuzz(3,  jazz_rules) == "Fizz"
```

---

### 6.3 Test doubles — mock, stub, spy, fake

Les test doubles sont des remplaçants d'objets réels utilisés dans les tests.
Il en existe plusieurs types, souvent confondus. Voici les définitions précises
de Gerard Meszaros (*xUnit Test Patterns*, 2007) :

| Type | Définition | Retourne des données | Vérifie les appels | Logique interne |
|---|---|:---:|:---:|:---:|
| **Dummy** | Objet passé en paramètre mais jamais utilisé | Non | Non | Non |
| **Stub** | Retourne des données préfixées, sans logique | Oui (fixes) | Non | Non |
| **Spy** | Enregistre les appels pour vérification ultérieure | Optionnel | Oui | Minimale |
| **Mock** | Vérifie les appels pendant ou après le test | Optionnel | Oui | Non |
| **Fake** | Implémentation simplifiée mais fonctionnelle | Oui | Non | Oui |

**Règle de choix :**

- Utilisez un **Stub** quand le test a besoin de données en entrée mais ne se
  soucie pas de qui les a produites.
- Utilisez un **Mock** quand le test vérifie qu'un comportement *a été appelé*
  (interaction testing).
- Utilisez un **Fake** quand plusieurs tests ont besoin d'un comportement
  cohérent (ex. : `InMemoryRepository`).
- **Evitez les Mocks sur les classes de votre propre domaine** : mocker une
  classe concrète interne crée un couplage entre le test et les détails
  d'implémentation. Mocker les interfaces/ports est acceptable.

```python
# ── Stub : retourne des données fixes ────────────────────────────────────────

class StubExchangeRateService:
    """Stub : retourne toujours le même taux, indépendamment de la date."""
    def get_rate(self, from_currency: str, to_currency: str) -> float:
        return 1.08  # EUR/USD fixe

def test_currency_converter_uses_exchange_rate():
    stub_service = StubExchangeRateService()
    converter    = CurrencyConverter(stub_service)
    assert converter.convert(100.0, "EUR", "USD") == pytest.approx(108.0)


# ── Fake : implémentation simplifiée mais fonctionnelle ──────────────────────

class InMemoryOrderRepository:
    """Fake : implémentation en mémoire pour les tests."""

    def __init__(self):
        self._store: dict[str, Order] = {}

    def save(self, order: Order) -> None:
        self._store[order.id.value] = order

    def find_by_id(self, order_id: OrderId) -> Order | None:
        return self._store.get(order_id.value)

    def find_all(self) -> list[Order]:
        return list(self._store.values())


# ── Mock : vérification des interactions ─────────────────────────────────────
from unittest.mock import MagicMock, call

def test_confirm_order_sends_notification():
    repo     = InMemoryOrderRepository()
    notifier = MagicMock()   # Mock auto-généré
    service  = OrderApplicationService(repo, notifier)

    order_id = service.create_order("customer-1", "product-42", quantity=3)
    service.confirm_order(order_id)

    # Vérification de l'interaction
    notifier.send_confirmation.assert_called_once()
    call_args = notifier.send_confirmation.call_args
    assert call_args.kwargs["customer_id"] == "customer-1"


# ── Spy : enregistrement pour vérification a posteriori ──────────────────────

class SpyLogger:
    def __init__(self):
        self.calls: list[dict] = []

    def info(self, message: str, **context) -> None:
        self.calls.append({"level": "INFO", "message": message, **context})

    def error(self, message: str, **context) -> None:
        self.calls.append({"level": "ERROR", "message": message, **context})

    def was_called_with_level(self, level: str) -> bool:
        return any(c["level"] == level for c in self.calls)

    def messages_containing(self, substring: str) -> list[str]:
        return [c["message"] for c in self.calls if substring in c["message"]]


def test_payment_failure_is_logged_as_error():
    spy_logger = SpyLogger()
    gateway    = MagicMock()
    gateway.charge.side_effect = ConnectionError("Timeout")

    service = PaymentService(gateway, spy_logger)
    service.process(amount=Decimal("50.00"), currency="EUR",
                    card_token="tok_test", customer_id="C001")

    assert spy_logger.was_called_with_level("ERROR")
    assert len(spy_logger.messages_containing("Erreur")) > 0
```

```php
<?php
// Stub en PHP avec PHPUnit
final class StubProductRepository implements ProductRepositoryInterface
{
    public function findById(ProductId $id): ?Product
    {
        return new Product($id, 'Widget', new Money(1999, 'EUR'));
    }

    public function save(Product $product): void { /* no-op */ }
}

// Mock avec PHPUnit
final class CartServiceTest extends TestCase
{
    public function testAddItemCallsRepositoryOnce(): void
    {
        $mockRepo = $this->createMock(ProductRepositoryInterface::class);
        $mockRepo->expects($this->once())
                 ->method('findById')
                 ->willReturn(new Product(
                     new ProductId('P1'), 'Widget', new Money(1999, 'EUR')
                 ));

        $service = new CartService($mockRepo);
        $service->addItem(new CartId('C1'), new ProductId('P1'), 2);
    }
}
```

---

### 6.4 Property-based testing

Les tests paramétriques classiques vérifient des exemples choisis manuellement.
Le **property-based testing** (PBT) renverse l'approche : on décrit des
*propriétés* qui doivent être vraies pour *toute* entrée valide, et le framework
génère automatiquement des centaines de cas de test aléatoires.

#### Principes

Au lieu de dire « `fizzbuzz(3) == "Fizz"` », on dit :
- « Pour tout multiple de 3, le résultat contient `"Fizz"` »
- « Pour tout entier dont le résultat est numérique, cet entier n'est divisible
  ni par 3, ni par 5, ni par 7 »

#### Hypothesis en Python

```python
# pip install hypothesis

from hypothesis import given, settings
from hypothesis import strategies as st


@given(st.integers(min_value=1, max_value=10_000))
def test_result_contains_fizz_iff_divisible_by_3(n: int):
    result = fizzbuzz(n)
    if n % 3 == 0:
        assert "Fizz" in result
    else:
        assert "Fizz" not in result


@given(st.integers(min_value=1, max_value=10_000))
def test_result_contains_buzz_iff_divisible_by_5(n: int):
    result = fizzbuzz(n)
    if n % 5 == 0:
        assert "Buzz" in result
    else:
        assert "Buzz" not in result


@given(st.integers(min_value=1, max_value=10_000))
def test_numeric_result_iff_not_divisible_by_3_5_or_7(n: int):
    result = fizzbuzz(n)
    is_numeric = result.lstrip("-").isdigit()
    should_be_numeric = n % 3 != 0 and n % 5 != 0 and n % 7 != 0
    assert is_numeric == should_be_numeric


# PBT sur les Value Objects : les invariants tiennent pour toute entrée
@given(
    st.decimals(min_value=0, max_value=1_000_000, places=2),
    st.decimals(min_value=0, max_value=1_000_000, places=2),
)
def test_money_addition_is_commutative(a, b):
    from decimal import Decimal
    m1 = Money(Decimal(str(a)), "EUR")
    m2 = Money(Decimal(str(b)), "EUR")
    assert m1.add(m2) == m2.add(m1)


@given(st.decimals(min_value=0, max_value=1_000_000, places=2))
def test_money_add_zero_is_identity(amount):
    from decimal import Decimal
    m    = Money(Decimal(str(amount)), "EUR")
    zero = Money(Decimal("0"), "EUR")
    assert m.add(zero) == m


# Round-trip property : sérialiser puis désérialiser retourne l'objet original
@given(st.integers(min_value=1), st.text(min_size=1, max_size=50))
def test_order_serialization_round_trip(order_id_int: int, customer_name: str):
    original   = {"id": order_id_int, "customer": customer_name}
    serialized = json.dumps(original)
    restored   = json.loads(serialized)
    assert restored == original
```

**Quand le PBT trouve un contre-exemple**, Hypothesis effectue un *shrinking* :
il simplifie automatiquement l'entrée fautive jusqu'au plus petit exemple qui
reproduit le bug. C'est extrêmement utile pour le débogage.

---

### 6.5 Tests d'architecture — vérifier le respect des couches

Les tests unitaires vérifient les comportements ; les tests d'architecture
vérifient que les **dépendances entre modules** respectent les règles
architecturales définies. Ils détectent automatiquement les violations de
la Clean Architecture ou du DDD (ex. : une classe du domaine qui importe
SQLAlchemy).

#### En Python — vérification manuelle avec `ast` et `importlib`

Python ne dispose pas d'un équivalent direct d'ArchUnit. Voici une approche
légère basée sur l'inspection des imports :

```python
# test_architecture.py

import ast
import os
import pytest
from pathlib import Path


def get_imports(filepath: str) -> set[str]:
    """Retourne tous les modules importés dans un fichier Python."""
    with open(filepath, "r", encoding="utf-8") as f:
        tree = ast.parse(f.read())
    imports = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                imports.add(alias.name.split(".")[0])
        elif isinstance(node, ast.ImportFrom):
            if node.module:
                imports.add(node.module.split(".")[0])
    return imports


def get_python_files(directory: str) -> list[str]:
    return [
        str(p) for p in Path(directory).rglob("*.py")
        if "__pycache__" not in str(p)
    ]


# Règle 1 : la couche domain n'importe jamais de modules d'infrastructure
INFRASTRUCTURE_MODULES = {"sqlalchemy", "pymongo", "redis", "boto3", "httpx", "requests"}

@pytest.mark.parametrize("filepath", get_python_files("src/domain"))
def test_domain_does_not_import_infrastructure(filepath: str):
    imports = get_imports(filepath)
    violations = imports & INFRASTRUCTURE_MODULES
    assert not violations, (
        f"{filepath} importe des modules d'infrastructure : {violations}\n"
        f"La couche domain ne doit dépendre que d'abstractions."
    )


# Règle 2 : la couche domain n'importe pas la couche application
@pytest.mark.parametrize("filepath", get_python_files("src/domain"))
def test_domain_does_not_import_application_layer(filepath: str):
    imports = get_imports(filepath)
    assert "application" not in imports, (
        f"{filepath} importe la couche application — violation de la règle de dépendance."
    )


# Règle 3 : tout Value Object doit être frozen (immuable)
@pytest.mark.parametrize("filepath", get_python_files("src/domain/value_objects"))
def test_value_objects_are_frozen_dataclasses(filepath: str):
    with open(filepath, "r") as f:
        content = f.read()
    tree = ast.parse(content)
    for node in ast.walk(tree):
        if isinstance(node, ast.ClassDef):
            for decorator in node.decorator_list:
                if isinstance(decorator, ast.Call):
                    if (hasattr(decorator.func, "id")
                            and decorator.func.id == "dataclass"):
                        for kw in decorator.keywords:
                            if kw.arg == "frozen":
                                assert kw.value.value, (
                                    f"Le Value Object {node.name} dans {filepath} "
                                    f"doit avoir @dataclass(frozen=True)."
                                )
```

#### En PHP — vérification avec PHPStan et règles custom

```php
<?php
// phpstan.neon — règles d'architecture via PHPStan

parameters:
    strictRules: true

rules:
    # Règle custom : les classes du namespace Domain ne doivent pas
    # dépendre du namespace Infrastructure
    - App\Tests\Architecture\DomainMustNotDependOnInfrastructureRule
```

```php
<?php
// DomainMustNotDependOnInfrastructureRule.php

use PhpParser\Node;
use PHPStan\Analyser\Scope;
use PHPStan\Rules\Rule;

/**
 * @implements Rule<Node\Stmt\Use_>
 */
final class DomainMustNotDependOnInfrastructureRule implements Rule
{
    public function getNodeType(): string
    {
        return Node\Stmt\Use_::class;
    }

    public function processNode(Node $node, Scope $scope): array
    {
        $currentNamespace = $scope->getNamespace() ?? '';

        // S'applique uniquement aux classes du namespace Domain
        if (!str_starts_with($currentNamespace, 'App\\Domain')) {
            return [];
        }

        $errors = [];
        foreach ($node->uses as $use) {
            $importedName = $use->name->toString();
            if (str_starts_with($importedName, 'App\\Infrastructure')) {
                $errors[] = sprintf(
                    'La couche Domain (%s) ne doit pas dépendre de Infrastructure (%s).',
                    $currentNamespace,
                    $importedName,
                );
            }
        }
        return $errors;
    }
}
```

---

### 6.6 Pyramide de tests vs Trophy de tests

#### La pyramide de tests (Mike Cohn, 2009)

```
         /\
        /  \
       / E2E\      Peu nombreux, lents, fragiles, coûteux
      /──────\
     / Intég. \    Modérés : testent les interactions entre composants
    /──────────\
   /  Unitaires \  Nombreux, rapides, isolés, peu coûteux
  /______________\
```

La pyramide recommande d'investir massivement dans les tests unitaires (base large),
modérément dans les tests d'intégration, et peu dans les tests end-to-end (pointe).

**Critiques :** la pyramide suppose que les tests unitaires sont nécessairement
isolés par des mocks. Si les mocks sont trop nombreux, les tests unitaires testent
des détails d'implémentation plutôt que des comportements — ils cassent à chaque
refactoring, même sans bug.

#### Le Trophy de tests (Kent C. Dodds, 2018)

```
         /\
        /E2E\        Quelques tests clés de bout en bout
       /──────\
      /Intégrat.\    Majorité des tests : testent les comportements réels
     /────────────\
    /   Unitaires  \ Tests purs (fonctions, Value Objects) — sans mocks
   /────────────────\
  /    Statique      \ Linting, typage statique, tests d'architecture
 /____________________\
```

Le Trophy met l'accent sur les **tests d'intégration** (au sens large : tests
qui exercent plusieurs classes ensemble sans infrastructure externe) et sur
l'**analyse statique** comme premier filet de sécurité.

**Recommandation pratique :**

- Pour les fonctions pures et les Value Objects → tests unitaires sans mocks.
- Pour les Application Services → tests avec Fakes (InMemoryRepository, etc.),
  sans mocks des classes internes.
- Pour les Repository, Adapters → tests d'intégration avec une vraie BDD de test
  (SQLite en mémoire, Docker).
- Pour les flux utilisateur critiques → quelques tests end-to-end.
- Pour toutes les couches → analyse statique (mypy, phpstan) et tests
  d'architecture.

---

### 6.7 Exercices du TD

#### Exercice A — Kata TDD : String Calculator

Implémentez une calculatrice de chaînes en appliquant strictement le cycle
Red / Green / Refactor. Chaque étape ne doit ajouter qu'un seul test et le
minimum de code pour le faire passer.

**Spécification :**

1. Une chaîne vide retourne `0`.
2. Un seul nombre retourne sa valeur : `"5"` → `5`.
3. Deux nombres séparés par une virgule retourne leur somme : `"1,2"` → `3`.
4. Un nombre quelconque de nombres est supporté : `"1,2,3,4"` → `10`.
5. Le séparateur peut aussi être un saut de ligne : `"1\n2,3"` → `6`.
6. Un séparateur custom peut être défini en première ligne : `"//;\n1;2"` → `3`.
7. Les nombres négatifs lèvent une exception avec la liste des négatifs :
   `"1,-2,-3"` → `NegativeNumberException: [-2, -3]`.
8. Les nombres supérieurs à 1000 sont ignorés : `"2,1001"` → `2`.

Contraintes :
- Ecrire un test qui échoue AVANT chaque nouvelle fonctionnalité.
- Le commit Git après chaque RED-GREEN-REFACTOR.
- Après la règle 6, refactoriser pour que l'ajout d'un nouveau séparateur
  ne nécessite pas de modifier le code existant.

---

#### Exercice B — Choisir le bon test double

Pour chaque scénario suivant, indiquez quel type de double utiliser (Stub, Mock,
Fake, Spy) et justifiez votre choix.

1. Tester que `InvoiceService.generate()` retourne une facture avec le bon total,
   en fournissant des données produit prédéfinies depuis le repository.

2. Tester que `PaymentService.process()` appelle bien `notifier.sendConfirmation()`
   exactement une fois en cas de succès.

3. Tester que l'ensemble du flux « créer un panier → ajouter des articles →
   calculer le total » fonctionne correctement, sans base de données.

4. Tester que `AuthService.login()` journalise les tentatives de connexion
   échouées avec le niveau `WARNING`.

5. Tester que `OrderExporter.exportToCsv()` produit un CSV valide pour dix
   commandes de structures différentes (Primitive Obsession vs Value Objects).

---

#### Exercice C — Property-based testing

En Python avec Hypothesis, écrivez les propriétés pour les spécifications
suivantes. Pour chaque propriété, formulez-la en langage naturel avant de
l'implémenter.

1. **Money :** `a.add(b) == b.add(a)` (commutativité), `a.add(Money(0, a.currency)) == a`
   (élément neutre), `a.add(b).amount == a.amount + b.amount` (conservation de la somme).

2. **DateRange :** une `DateRange` valide a toujours une durée positive.
   Deux `DateRange` qui se chevauchent vérifient `range1.overlaps(range2) == range2.overlaps(range1)`.

3. **Either / Railway :** si toutes les étapes d'un pipeline réussissent,
   le résultat final est un `Right`. Si au moins une étape échoue, le résultat
   est un `Left` et le message d'erreur correspond à l'étape échouée.

---

## Récapitulatif de la matinée

| Concept | Ce qu'il apporte | Lien avec la formation |
|---|---|---|
| Fonctions pures | Testabilité triviale, raisonnement local | Complement des Value Objects DDD (session 4) |
| Immutabilité | Absence de bugs liés à l'état partagé | Les Value Objects DDD sont déjà immuables |
| HOF / composition | Pipelines de transformation sans duplication | Alternative fonctionnelle au pattern Template Method |
| Maybe / Option | Absence de valeur explicite dans le type | Elimine les NullPointerException silencieux |
| Either / Railway | Erreurs métier explicites, pas d'exceptions de flux | Complément à la validation dans les Aggregates |
| Cycle TDD | Code testable par construction, design émergent | Complète le refactoring guidé (session 2) |
| Test doubles | Isolation des unités testées | Permet de tester les Application Services sans infrastructure |
| Property-based testing | Exploration de l'espace des entrées, invariants | Vérifie les Value Objects et les fonctions pures exhaustivement |
| Tests d'architecture | Détection automatique des violations de couches | Renforce la séparation DDD domaine / infrastructure |
| Pyramide / Trophy | Stratégie globale de couverture | Cadre pour décider quoi tester à chaque niveau |

L'après-midi clôt la formation en montrant comment tous ces éléments s'assemblent
dans une architecture évolutive : hexagonale, Clean Architecture, CQRS et les
fondements de l'Event Sourcing.

---

## Bibliographie

### Test-Driven Development

**Beck, K.** (2002). *Test-Driven Development: By Example*. Addison-Wesley.  
La référence fondatrice du TDD, écrite par son inventeur. Le livre se lit en suivant
deux exemples fil rouge (une implémentation de Money en Java, un framework de test en
Python) construits pas à pas selon le cycle Red/Green/Refactor. Les cent premières pages
suffisent pour saisir l'essentiel ; la troisième partie (patterns TDD) approfondit les
cas délicats. Ouvrage court, dense et très concret — à lire en ayant un IDE ouvert.

**Freeman, S. & Pryce, N.** (2009). *Growing Object-Oriented Software, Guided by Tests*.
Addison-Wesley.  
Le livre qui articule le mieux TDD et conception orientée objet. Son apport principal :
montrer que le TDD fait émerger une architecture hexagonale naturellement, en commençant
par un test de bout en bout et en faisant croître le système par l'intérieur. Les chapitres
4 à 6 (kickstart d'un projet en TDD) et 19 à 22 (concevoir avec des mocks) correspondent
directement au contenu de la session 6. Souvent cité comme le complément indispensable
du livre de Beck pour les développeurs orientés objet.

### Test doubles

**Meszaros, G.** (2007). *xUnit Test Patterns: Refactoring Test Code*. Addison-Wesley.  
L'ouvrage qui établit la taxonomie de référence des test doubles présentée en session 6 :
Dummy, Stub, Spy, Mock, Fake — avec leurs définitions précises et leurs cas d'usage.
Volumineux (880 pages), il est surtout utile comme dictionnaire de référence plutôt que
comme lecture linéaire. La partie II (les smells de code de test) et le catalogue des
patterns (partie III) sont les sections les plus utiles au quotidien.

**Fowler, M.** (2007). Mocks Aren't Stubs. *MartinFowler.com*.  
Article de référence qui distingue les approches « classique » et « mockiste » du TDD,
et explique pourquoi confondre Mock et Stub mène à des tests fragiles. Court (15 minutes
de lecture), il clarifie la taxonomie de Meszaros avec des exemples concrets.
https://martinfowler.com/articles/mocksArentStubs.html

### Programmation fonctionnelle

**Wlaschin, S.** (2018). *Domain Modeling Made Functional*. Pragmatic Bookshelf.  
Le livre qui articule le plus clairement DDD et programmation fonctionnelle. Rédigé en
F#, ses concepts se transposent directement en Python ou TypeScript. Le chapitre 6
(intégrité et cohérence des données par les types) et les chapitres 10 et 11 (Railway
Oriented Programming et gestion des erreurs) correspondent précisément au contenu de la
session 5. Même sans connaître F#, la lecture reste accessible : l'auteur introduit le
langage au fil du texte.

**Wlaschin, S.** (2013). Railway Oriented Programming. *F# for Fun and Profit*.  
L'article de blog originel qui popularise le Railway Oriented Programming. Disponible
avec des diagrammes animés sur https://fsharpforfun.com/posts/recipe-part2.html.
Complémentaire du livre, il expose la métaphore ferroviaire de façon très visuelle et
peut être montré directement en cours comme support illustratif.

**Hutton, G.** (2016). *Programming in Haskell* (2e éd.). Cambridge University Press.  
Pour les étudiants qui souhaitent approfondir les fondements théoriques des concepts
présentés (monades, foncteurs, applicatives). Haskell reste la langue maternelle de la
programmation fonctionnelle pure ; comprendre ses abstractions éclaire rétrospectivement
`Maybe`, `Either` et la composition de fonctions dans n'importe quel langage. Les
chapitres 1 à 7 suffisent pour cette ambition.

### Property-based testing

**Claessen, K. & Hughes, J.** (2000). QuickCheck: A Lightweight Tool for Random Testing
of Haskell Programs. *ACM SIGPLAN Notices*, 35(9), 268–279.  
L'article académique qui a introduit le property-based testing. En huit pages, il pose
les bases du générateur de données aléatoires, du shrinking et de la formulation de
propriétés — concepts que tous les frameworks PBT modernes (Hypothesis, fast-check,
jqwik) ont repris. Disponible gratuitement sur le site de John Hughes.

**MacIver, D.** Documentation de *Hypothesis*. https://hypothesis.readthedocs.io  
Hypothesis est la bibliothèque Python de property-based testing présentée en session 6.
Sa documentation contient un tutorial complet, un guide des stratégies de génération de
données et des exemples de propriétés réalistes. La section « What is Hypothesis? » et
le tutorial en cinq minutes constituent une lecture préparatoire suffisante pour l'atelier.

### Stratégie de tests

**Cohn, M.** (2009). *Succeeding with Agile: Software Development Using Scrum*. Addison-Wesley.  
Ouvrage qui introduit la pyramide de tests dans sa formulation la plus connue. Le
chapitre 16 (testing) présente les trois niveaux (unitaires, de service, d'interface)
et leur équilibre recommandé. Incontournable pour positionner chaque type de test dans
une stratégie globale.

**Dodds, K. C.** (2018). Write Tests. Not Too Many. Mostly Integration. *kentcdodds.com*.  
Article de blog qui introduit le Trophy de tests comme alternative à la pyramide, en
argumentant que les tests d'intégration (au sens large) offrent le meilleur rapport
confiance / coût de maintenance. Court et provocateur, il fait un bon sujet de débat
en fin de session pour discuter des mérites comparatifs des deux modèles.
https://kentcdodds.com/blog/write-tests
