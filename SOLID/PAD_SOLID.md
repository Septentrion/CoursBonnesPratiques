# Principes SOLID — Support de cours

**Public cible :** Etudiants en informatique ayant déjà suivi un cours d'introduction aux design patterns.  
**Langages illustrés :** PHP 8.x et Python 3.10+  
**Objectif :** Comprendre, reconnaître et appliquer les cinq principes SOLID dans des contextes réalistes.

---

## Introduction

Les principes SOLID ont été formulés par Robert C. Martin (« Uncle Bob ») au tournant des années 2000 et popularisés dans son ouvrage *Agile Software Development, Principles, Patterns, and Practices* (2002). Ils constituent un socle de référence pour concevoir du code orienté objet maintenable, extensible et testable.

L'acronyme regroupe cinq principes indépendants mais complémentaires :

| Lettre | Principe | Formulation courte |
|--------|----------|--------------------|
| **S** | Single Responsibility Principle | Une classe, une seule raison de changer. |
| **O** | Open/Closed Principle | Ouvert à l'extension, fermé à la modification. |
| **L** | Liskov Substitution Principle | Un sous-type doit pouvoir remplacer son type de base. |
| **I** | Interface Segregation Principle | Mieux vaut plusieurs interfaces spécifiques qu'une seule interface générale. |
| **D** | Dependency Inversion Principle | Dépendre des abstractions, pas des concretions. |

Ces principes ne sont pas des règles absolues mais des **heuristiques de conception** : leur application doit toujours être proportionnée à la complexité réelle du problème.

---

## 1. Single Responsibility Principle (SRP)

### 1.1 Définition

> Une classe ne devrait avoir qu'une seule raison de changer.

Autrement dit, une classe doit être responsable d'**un seul acteur** (au sens de Robert C. Martin : un groupe de personnes ou un système qui peut exiger un changement). La confusion entre plusieurs responsabilités entraîne un couplage accidentel : modifier le comportement d'un acteur risque d'en casser un autre.

Le SRP ne signifie pas qu'une classe ne doit faire qu'une seule chose, mais qu'elle ne doit servir qu'un seul maître.

### 1.2 Exemple PHP

**Violation du SRP**

La classe `UserManager` mélange trois responsabilités distinctes : la logique métier utilisateur, la persistence en base de données et l'envoi d'e-mails.

```php
<?php

class UserManager
{
    private PDO $pdo;

    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }

    public function createUser(string $name, string $email): void
    {
        // Responsabilité 1 : logique métier
        if (empty($name) || !filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException('Données utilisateur invalides.');
        }

        // Responsabilité 2 : persistance
        $stmt = $this->pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
        $stmt->execute([$name, $email]);

        // Responsabilité 3 : notification par e-mail
        mail($email, 'Bienvenue', "Bonjour $name, votre compte a été créé.");
    }
}
```

**Application du SRP**

On sépare les trois responsabilités en classes distinctes, chacune avec une seule raison de changer.

```php
<?php

// Responsabilité 1 : validation et représentation métier
class User
{
    public function __construct(
        public readonly string $name,
        public readonly string $email
    ) {
        if (empty($name)) {
            throw new \InvalidArgumentException('Le nom ne peut pas être vide.');
        }
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("L'adresse e-mail est invalide.");
        }
    }
}

// Responsabilité 2 : persistance
class UserRepository
{
    public function __construct(private readonly PDO $pdo) {}

    public function save(User $user): void
    {
        $stmt = $this->pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
        $stmt->execute([$user->name, $user->email]);
    }
}

// Responsabilité 3 : notification
class UserNotifier
{
    public function sendWelcome(User $user): void
    {
        mail($user->email, 'Bienvenue', "Bonjour {$user->name}, votre compte a été créé.");
    }
}

// Orchestration dans un service applicatif
class UserRegistrationService
{
    public function __construct(
        private readonly UserRepository $repository,
        private readonly UserNotifier   $notifier
    ) {}

    public function register(string $name, string $email): void
    {
        $user = new User($name, $email);
        $this->repository->save($user);
        $this->notifier->sendWelcome($user);
    }
}
```

### 1.3 Exemple Python

**Violation du SRP**

```python
import sqlite3
import smtplib
from email.message import EmailMessage


class ReportManager:
    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)

    def generate(self, user_id: int) -> dict:
        # Responsabilité 1 : récupération des données
        cursor = self.conn.execute(
            "SELECT name, email, orders FROM users WHERE id = ?", (user_id,)
        )
        row = cursor.fetchone()
        if row is None:
            raise ValueError(f"Utilisateur {user_id} introuvable.")

        # Responsabilité 2 : calcul / logique métier
        name, email, orders = row
        summary = {"name": name, "email": email, "total_orders": orders}

        # Responsabilité 3 : envoi par e-mail
        msg = EmailMessage()
        msg["Subject"] = "Votre rapport mensuel"
        msg["From"] = "noreply@example.com"
        msg["To"] = email
        msg.set_content(f"Bonjour {name}, vous avez passé {orders} commandes.")
        with smtplib.SMTP("localhost") as smtp:
            smtp.send_message(msg)

        return summary
```

**Application du SRP**

```python
import sqlite3
import smtplib
from dataclasses import dataclass
from email.message import EmailMessage


@dataclass
class UserReport:
    name: str
    email: str
    total_orders: int


# Responsabilité 1 : accès aux données
class ReportRepository:
    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)

    def fetch_user_report(self, user_id: int) -> UserReport:
        cursor = self.conn.execute(
            "SELECT name, email, orders FROM users WHERE id = ?", (user_id,)
        )
        row = cursor.fetchone()
        if row is None:
            raise ValueError(f"Utilisateur {user_id} introuvable.")
        return UserReport(name=row[0], email=row[1], total_orders=row[2])


# Responsabilité 2 : génération du contenu du rapport
class ReportFormatter:
    def format(self, report: UserReport) -> str:
        return (
            f"Bonjour {report.name}, "
            f"vous avez passé {report.total_orders} commandes ce mois-ci."
        )


# Responsabilité 3 : envoi de l'e-mail
class ReportMailer:
    def send(self, report: UserReport, body: str) -> None:
        msg = EmailMessage()
        msg["Subject"] = "Votre rapport mensuel"
        msg["From"] = "noreply@example.com"
        msg["To"] = report.email
        msg.set_content(body)
        with smtplib.SMTP("localhost") as smtp:
            smtp.send_message(msg)


# Orchestration
class ReportService:
    def __init__(
        self,
        repository: ReportRepository,
        formatter: ReportFormatter,
        mailer: ReportMailer,
    ):
        self.repository = repository
        self.formatter = formatter
        self.mailer = mailer

    def send_monthly_report(self, user_id: int) -> None:
        report = self.repository.fetch_user_report(user_id)
        body = self.formatter.format(report)
        self.mailer.send(report, body)
```

### 1.4 Exercices

**Exercice 1 — Niveau débutant**

La classe suivante gère à la fois la lecture d'un fichier CSV et le calcul de statistiques :

```php
<?php
class CsvAnalyzer
{
    public function analyze(string $filepath): array
    {
        $lines = file($filepath, FILE_IGNORE_NEW_LINES);
        $values = array_map('floatval', $lines);
        $mean = array_sum($values) / count($values);
        $max  = max($values);
        $min  = min($values);
        // Ecriture du résultat dans un fichier
        file_put_contents('result.txt', "Moy: $mean, Max: $max, Min: $min");
        return compact('mean', 'max', 'min');
    }
}
```

- Identifiez les responsabilités présentes dans cette classe.
- Refactorisez en créant autant de classes que de responsabilités distinctes.
- Vérifiez que chaque classe obtenue n'a bien qu'une seule raison de changer ; formulez cette raison en une phrase.

---

**Exercice 2 — Niveau intermédiaire**

En Python, on vous soumet la classe `OrderProcessor` qui :
- valide les données d'une commande (montant positif, produit existant) ;
- applique une réduction selon un code promo ;
- insère la commande dans une base SQLite ;
- génère une facture au format PDF via une bibliothèque externe ;
- envoie la facture par e-mail au client.

Concevez le découpage en classes/modules respectant le SRP. Pour chaque classe, précisez :
- sa responsabilité unique ;
- les dépendances qu'elle reçoit (injection de dépendances) ;
- les méthodes publiques qu'elle expose.

Il n'est pas nécessaire d'implémenter le code complet ; un diagramme de classes textuel (ou pseudo-code) est suffisant.

---

**Exercice 3 — Niveau avancé**

Vous reprenez une application legacy PHP de gestion de blog. La classe centrale `PostController` contient actuellement :
- le routage HTTP (analyse de `$_GET` et `$_POST`) ;
- la validation des données du formulaire ;
- l'accès direct à la base de données via PDO ;
- le rendu HTML par concaténation de chaînes ;
- la gestion du cache via les en-têtes HTTP ;
- la journalisation des erreurs dans un fichier texte.

Votre mission :
1. Listez les acteurs (parties prenantes) à l'origine de chaque responsabilité.
2. Proposez une architecture en couches (Controller, Service, Repository, View) qui respecte le SRP et qui soit testable unitairement.
3. Implémentez la couche Service (`PostService`) et la couche Repository (`PostRepository`) en PHP, en vous limitant aux méthodes `findById`, `findAll`, `save` et `delete`.
4. Expliquez comment votre découpage facilite le remplacement futur du moteur de base de données (SQLite → PostgreSQL) sans toucher à la couche Service.

---

## 2. Open/Closed Principle (OCP)

### 2.1 Définition

> Les entités logicielles doivent être ouvertes à l'extension mais fermées à la modification.

Formulé à l'origine par Bertrand Meyer (1988) et réinterprété par Robert C. Martin, ce principe indique qu'ajouter un nouveau comportement ne devrait pas nécessiter de modifier le code existant — seulement d'y ajouter du code nouveau (via l'héritage, la composition ou des stratégies interchangeables).

En pratique, l'OCP est souvent atteint par la combinaison du **polymorphisme** et de l'**injection de dépendances**.

### 2.2 Exemple PHP

**Violation de l'OCP**

Chaque nouveau type de remise nécessite de modifier la méthode `apply` par un nouveau `if`.

```php
<?php

class DiscountCalculator
{
    public function apply(float $price, string $discountType): float
    {
        if ($discountType === 'percentage') {
            return $price * 0.90; // 10 % de remise
        }

        if ($discountType === 'fixed') {
            return $price - 5.00;
        }

        if ($discountType === 'blackfriday') {
            return $price * 0.50;
        }

        return $price;
    }
}
```

**Application de l'OCP**

On délègue le calcul à une stratégie polymorphe. Ajouter un nouveau type de remise revient à créer une nouvelle classe, sans toucher aux existantes.

```php
<?php

interface DiscountStrategy
{
    public function apply(float $price): float;
}

class PercentageDiscount implements DiscountStrategy
{
    public function __construct(private readonly float $rate) {}

    public function apply(float $price): float
    {
        return $price * (1 - $this->rate);
    }
}

class FixedDiscount implements DiscountStrategy
{
    public function __construct(private readonly float $amount) {}

    public function apply(float $price): float
    {
        return max(0, $price - $this->amount);
    }
}

class BlackFridayDiscount implements DiscountStrategy
{
    public function apply(float $price): float
    {
        return $price * 0.50;
    }
}

// Ajout futur sans modifier le code ci-dessus :
class LoyaltyDiscount implements DiscountStrategy
{
    public function __construct(private readonly int $loyaltyPoints) {}

    public function apply(float $price): float
    {
        $reduction = min($this->loyaltyPoints * 0.01, $price * 0.20);
        return $price - $reduction;
    }
}

class OrderPricer
{
    public function __construct(private readonly DiscountStrategy $discount) {}

    public function computeTotal(float $basePrice): float
    {
        return $this->discount->apply($basePrice);
    }
}
```

### 2.3 Exemple Python

**Violation de l'OCP**

```python
class ReportExporter:
    def export(self, data: list[dict], format: str) -> str:
        if format == "csv":
            header = ",".join(data[0].keys())
            rows = [",".join(str(v) for v in row.values()) for row in data]
            return header + "\n" + "\n".join(rows)

        if format == "json":
            import json
            return json.dumps(data, ensure_ascii=False, indent=2)

        if format == "html":
            rows_html = "".join(
                "<tr>" + "".join(f"<td>{v}</td>" for v in row.values()) + "</tr>"
                for row in data
            )
            return f"<table>{rows_html}</table>"

        raise ValueError(f"Format inconnu : {format}")
```

**Application de l'OCP**

```python
from abc import ABC, abstractmethod
import json


class ExportStrategy(ABC):
    @abstractmethod
    def export(self, data: list[dict]) -> str:
        ...


class CsvExporter(ExportStrategy):
    def export(self, data: list[dict]) -> str:
        if not data:
            return ""
        header = ",".join(data[0].keys())
        rows = [",".join(str(v) for v in row.values()) for row in data]
        return header + "\n" + "\n".join(rows)


class JsonExporter(ExportStrategy):
    def export(self, data: list[dict]) -> str:
        return json.dumps(data, ensure_ascii=False, indent=2)


class HtmlExporter(ExportStrategy):
    def export(self, data: list[dict]) -> str:
        rows_html = "".join(
            "<tr>" + "".join(f"<td>{v}</td>" for v in row.values()) + "</tr>"
            for row in data
        )
        return f"<table>{rows_html}</table>"


# Nouvel export ajouté sans modifier l'existant :
class MarkdownExporter(ExportStrategy):
    def export(self, data: list[dict]) -> str:
        if not data:
            return ""
        headers = list(data[0].keys())
        sep = " | ".join("---" for _ in headers)
        header_row = " | ".join(headers)
        rows = [" | ".join(str(row[h]) for h in headers) for row in data]
        return "\n".join([header_row, sep] + rows)


class ReportExporter:
    def __init__(self, strategy: ExportStrategy):
        self.strategy = strategy

    def export(self, data: list[dict]) -> str:
        return self.strategy.export(data)
```

### 2.4 Exercices

**Exercice 1 — Niveau débutant**

Vous disposez du code PHP suivant :

```php
<?php
class ShippingCostCalculator
{
    public function calculate(string $carrier, float $weight): float
    {
        if ($carrier === 'colissimo') {
            return 3.50 + $weight * 0.80;
        }
        if ($carrier === 'chronopost') {
            return 7.00 + $weight * 1.20;
        }
        return 0;
    }
}
```

- Refactorisez ce code pour respecter l'OCP.
- Ajoutez ensuite le transporteur `DHL` (tarif : 9,00 € + 1,50 €/kg) **sans modifier** les classes existantes.
- Justifiez en quoi votre solution est désormais fermée à la modification.

---

**Exercice 2 — Niveau intermédiaire**

En Python, concevez un système de validation de formulaire extensible :
- La classe `FormValidator` doit appliquer une liste de règles de validation sur un dictionnaire de données.
- Chaque règle (obligatoire, longueur minimale, format e-mail, valeur numérique, etc.) est une classe indépendante.
- L'ajout d'une nouvelle règle (`CreditCardRule`, `PhoneNumberRule`...) ne doit pas nécessiter de modifier `FormValidator`.
- Implémentez au moins quatre règles concrètes et un test démontrant leur composition.

---

**Exercice 3 — Niveau avancé**

Vous concevez un pipeline de traitement d'images en PHP. Chaque étape (redimensionnement, conversion en niveaux de gris, filigrane, compression) est un filtre indépendant.

1. Définissez une interface `ImageFilter` et implémentez quatre filtres concrets.
2. Implémentez une classe `ImagePipeline` qui enchaîne les filtres dans l'ordre donné à la construction.
3. Démontrez qu'il est possible d'ajouter un filtre `AiEnhancementFilter` (appel à une API externe simulée) sans modifier `ImagePipeline` ni aucun filtre existant.
4. Discutez les limites de l'OCP : dans quels cas la modification du code existant est-elle inévitable ou préférable à l'extension ?

---

## 3. Liskov Substitution Principle (LSP)

### 3.1 Définition

> Les objets d'un programme doivent pouvoir être remplacés par des instances de leurs sous-types sans altérer la correction du programme.

Formulé par Barbara Liskov (1987), ce principe impose des contraintes précises sur la relation d'héritage :

- **Préconditions** : un sous-type ne peut pas renforcer les préconditions (exiger plus que le parent).
- **Postconditions** : un sous-type ne peut pas affaiblir les postconditions (garantir moins que le parent).
- **Invariants** : un sous-type doit préserver les invariants de la classe parente.
- **Exceptions** : un sous-type ne peut pas lever de nouvelles exceptions non prévues par le contrat parent.

La violation la plus classique est le carré-rectangle : un `Square` qui hérite de `Rectangle` et redéfinit `setWidth` pour modifier aussi `setHeight` viole le LSP.

### 3.2 Exemple PHP

**Violation du LSP**

```php
<?php

class Rectangle
{
    protected int $width;
    protected int $height;

    public function setWidth(int $width): void  { $this->width  = $width; }
    public function setHeight(int $height): void { $this->height = $height; }
    public function area(): int { return $this->width * $this->height; }
}

class Square extends Rectangle
{
    // Un carré impose width == height : on surcharge les deux setters.
    public function setWidth(int $width): void
    {
        $this->width  = $width;
        $this->height = $width; // effet de bord inattendu
    }

    public function setHeight(int $height): void
    {
        $this->height = $height;
        $this->width  = $height; // idem
    }
}

// Ce code fonctionne pour Rectangle mais pas pour Square :
function assertArea(Rectangle $r): void
{
    $r->setWidth(4);
    $r->setHeight(5);
    assert($r->area() === 20, "Aire attendue : 20"); // echoue avec Square (25)
}
```

**Application du LSP**

On abandonne l'héritage `Square extends Rectangle` et on utilise une hiérarchie basée sur une abstraction commune.

```php
<?php

interface Shape
{
    public function area(): float;
}

class Rectangle implements Shape
{
    public function __construct(
        private float $width,
        private float $height
    ) {}

    public function area(): float
    {
        return $this->width * $this->height;
    }
}

class Square implements Shape
{
    public function __construct(private float $side) {}

    public function area(): float
    {
        return $this->side ** 2;
    }
}

// Le code client travaille sur l'abstraction Shape :
function printArea(Shape $shape): void
{
    echo "Aire : " . $shape->area() . PHP_EOL;
}
```

### 3.3 Exemple Python

**Violation du LSP**

```python
class Bird:
    def fly(self) -> str:
        return "Je vole."


class Penguin(Bird):
    def fly(self) -> str:
        # Un pingouin ne vole pas : on lève une exception non prévue
        # par le contrat de Bird.
        raise NotImplementedError("Les pingouins ne volent pas.")


def make_bird_fly(bird: Bird) -> None:
    # Ce code suppose que tout Bird peut voler — contrat brisé par Penguin.
    print(bird.fly())
```

**Application du LSP**

```python
from abc import ABC, abstractmethod


class Animal(ABC):
    @abstractmethod
    def move(self) -> str:
        ...


class FlyingBird(Animal):
    @abstractmethod
    def fly(self) -> str:
        ...

    def move(self) -> str:
        return self.fly()


class SwimmingBird(Animal):
    @abstractmethod
    def swim(self) -> str:
        ...

    def move(self) -> str:
        return self.swim()


class Sparrow(FlyingBird):
    def fly(self) -> str:
        return "Le moineau bat des ailes."


class Eagle(FlyingBird):
    def fly(self) -> str:
        return "L'aigle plane en altitude."


class Penguin(SwimmingBird):
    def swim(self) -> str:
        return "Le pingouin nage sous l'eau."


def describe_movement(animal: Animal) -> None:
    # Fonctionne correctement quel que soit le sous-type.
    print(animal.move())
```

### 3.4 Exercices

**Exercice 1 — Niveau débutant**

Analysez la hiérarchie PHP suivante et identifiez la ou les violations du LSP :

```php
<?php
class Account
{
    protected float $balance = 0;

    public function deposit(float $amount): void
    {
        if ($amount <= 0) throw new \InvalidArgumentException('Montant invalide.');
        $this->balance += $amount;
    }

    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) throw new \RuntimeException('Solde insuffisant.');
        $this->balance -= $amount;
    }

    public function getBalance(): float { return $this->balance; }
}

class ReadOnlyAccount extends Account
{
    public function deposit(float $amount): void
    {
        throw new \RuntimeException('Ce compte est en lecture seule.');
    }

    public function withdraw(float $amount): void
    {
        throw new \RuntimeException('Ce compte est en lecture seule.');
    }
}
```

- Expliquez pourquoi `ReadOnlyAccount` viole le LSP.
- Proposez une refactorisation qui respecte le principe tout en modélisant correctement un compte en lecture seule.

---

**Exercice 2 — Niveau intermédiaire**

En Python, vous disposez d'un système de stockage de fichiers :

```python
class FileStorage:
    def read(self, path: str) -> bytes: ...
    def write(self, path: str, data: bytes) -> None: ...
    def delete(self, path: str) -> None: ...
    def list_files(self) -> list[str]: ...
```

On souhaite ajouter `S3Storage`, `ReadOnlyLocalStorage` et `EncryptedStorage`.

- Identifiez quelles sous-classes risquent de violer le LSP si elles héritent directement de `FileStorage`.
- Concevez une hiérarchie d'interfaces (ou classes abstraites) qui satisfait le LSP pour chaque cas.
- Implémentez `ReadOnlyLocalStorage` de manière conforme.

---

**Exercice 3 — Niveau avancé**

Vous concevez un framework de serialisation PHP. La classe de base `Serializer` expose :
- `serialize(object $obj): string`
- `deserialize(string $data, string $class): object`
- `supportsFormat(string $format): bool`

On veut créer : `JsonSerializer`, `XmlSerializer`, `BinarySerializer` et `WriteOnlySerializer` (qui ne supporte pas la désérialisation).

1. Déterminez les pré- et post-conditions de chaque méthode pour formaliser le contrat de `Serializer`.
2. Identifiez le problème posé par `WriteOnlySerializer` au regard du LSP.
3. Refactorisez la hiérarchie pour que chaque classe concrète satisfasse rigoureusement le LSP. Vous pouvez introduire autant d'interfaces que nécessaire.
4. Expliquez le lien entre le LSP et le principe ISP (que vous verrez à la section suivante).

---

## 4. Interface Segregation Principle (ISP)

### 4.1 Définition

> Les clients ne doivent pas être forcés de dépendre d'interfaces qu'ils n'utilisent pas.

Ce principe, introduit par Robert C. Martin, vise à éviter les « interfaces obèses » qui regroupent de nombreux contrats sans lien fort entre eux. Une classe qui implémente une interface trop large est contrainte de fournir des méthodes vides ou levant des exceptions, ce qui est un signal fort de violation.

L'ISP conduit naturellement à des interfaces cohésives et à l'utilisation de la composition d'interfaces plutôt qu'à une hiérarchie monolithique.

### 4.2 Exemple PHP

**Violation de l'ISP**

```php
<?php

// Interface trop large : pas tous les animaux nagent, volent et courent.
interface Animal
{
    public function eat(): void;
    public function sleep(): void;
    public function fly(): void;
    public function swim(): void;
    public function run(): void;
}

class Dog implements Animal
{
    public function eat(): void  { echo "Le chien mange."; }
    public function sleep(): void { echo "Le chien dort."; }
    public function run(): void  { echo "Le chien court."; }
    public function swim(): void { echo "Le chien nage."; }

    // Méthode inutile : un chien ne vole pas.
    public function fly(): void
    {
        throw new \LogicException('Les chiens ne volent pas.');
    }
}
```

**Application de l'ISP**

```php
<?php

interface CanEat   { public function eat(): void; }
interface CanSleep { public function sleep(): void; }
interface CanFly   { public function fly(): void; }
interface CanSwim  { public function swim(): void; }
interface CanRun   { public function run(): void; }

class Dog implements CanEat, CanSleep, CanSwim, CanRun
{
    public function eat(): void   { echo "Le chien mange."; }
    public function sleep(): void { echo "Le chien dort."; }
    public function swim(): void  { echo "Le chien nage."; }
    public function run(): void   { echo "Le chien court."; }
    // Pas de fly() : le contrat ne l'exige pas.
}

class Eagle implements CanEat, CanSleep, CanFly
{
    public function eat(): void   { echo "L'aigle mange."; }
    public function sleep(): void { echo "L'aigle dort."; }
    public function fly(): void   { echo "L'aigle vole."; }
}

class Dolphin implements CanEat, CanSleep, CanSwim
{
    public function eat(): void   { echo "Le dauphin mange."; }
    public function sleep(): void { echo "Le dauphin dort."; }
    public function swim(): void  { echo "Le dauphin nage."; }
}

// Le code client ne dépend que de ce dont il a besoin :
function makeItFly(CanFly $animal): void
{
    $animal->fly();
}
```

### 4.3 Exemple Python

**Violation de l'ISP**

```python
from abc import ABC, abstractmethod


class WorkerInterface(ABC):
    @abstractmethod
    def work(self) -> None: ...

    @abstractmethod
    def eat(self) -> None: ...

    @abstractmethod
    def sleep(self) -> None: ...

    @abstractmethod
    def attend_meeting(self) -> None: ...


class Robot(WorkerInterface):
    def work(self) -> None:
        print("Le robot travaille.")

    # Un robot ne mange pas ni ne dort ni n'assiste à des réunions.
    def eat(self) -> None:
        raise NotImplementedError("Les robots ne mangent pas.")

    def sleep(self) -> None:
        raise NotImplementedError("Les robots ne dorment pas.")

    def attend_meeting(self) -> None:
        raise NotImplementedError("Les robots n'assistent pas aux réunions.")
```

**Application de l'ISP**

```python
from abc import ABC, abstractmethod


class Workable(ABC):
    @abstractmethod
    def work(self) -> None: ...


class Feedable(ABC):
    @abstractmethod
    def eat(self) -> None: ...


class Sleepable(ABC):
    @abstractmethod
    def sleep(self) -> None: ...


class Meetable(ABC):
    @abstractmethod
    def attend_meeting(self) -> None: ...


class HumanEmployee(Workable, Feedable, Sleepable, Meetable):
    def work(self) -> None:
        print("L'employé travaille.")

    def eat(self) -> None:
        print("L'employé déjeune.")

    def sleep(self) -> None:
        print("L'employé se repose.")

    def attend_meeting(self) -> None:
        print("L'employé participe à la réunion.")


class Robot(Workable):
    def work(self) -> None:
        print("Le robot travaille sans interruption.")


# Le code client n'impose que le minimum nécessaire :
def run_production_line(workers: list[Workable]) -> None:
    for worker in workers:
        worker.work()
```

### 4.4 Exercices

**Exercice 1 — Niveau débutant**

L'interface PHP suivante est utilisée dans une application d'e-commerce :

```php
<?php
interface ProductRepository
{
    public function findById(int $id): ?array;
    public function findAll(): array;
    public function save(array $product): void;
    public function delete(int $id): void;
    public function findByCategory(string $category): array;
    public function updateStock(int $id, int $quantity): void;
    public function generatePriceReport(): string;
    public function exportToCsv(): string;
}
```

- Identifiez les groupes de responsabilités présents dans cette interface.
- Découpez-la en interfaces cohésives selon l'ISP.
- Créez deux classes concrètes `InMemoryProductRepository` (pour les tests) et `DatabaseProductRepository` qui implémentent uniquement les interfaces pertinentes.

---

**Exercice 2 — Niveau intermédiaire**

Concevez en Python un système de plugins pour un éditeur de texte. Un plugin peut offrir une ou plusieurs des capacités suivantes :
- Analyse syntaxique (`syntax_check(code: str) -> list[str]`)
- Autocomplétion (`suggest(prefix: str) -> list[str]`)
- Formatage (`format(code: str) -> str`)
- Débogage (`debug(code: str) -> str`)

Certains plugins (ex : `LightweightPlugin`) ne proposent que le formatage. D'autres (ex : `FullIdePlugin`) les offrent toutes.

- Définissez les interfaces ségrégées.
- Implémentez `LightweightPlugin`, `FullIdePlugin` et `LinterPlugin` (analyse + formatage uniquement).
- Ecrivez une fonction `run_formatter(plugin)` qui accepte n'importe quel plugin capable de formater.

---

**Exercice 3 — Niveau avancé**

Vous développez une bibliothèque PHP de gestion de médias (images, vidéos, audio). Partant d'une interface `Media` unique contenant vingt méthodes, votre mission est de :

1. Analyser les méthodes et les regrouper par capacité (lecture, écriture, transcodage, métadonnées, streaming, miniatures...).
2. Définir une interface par capacité, avec une documentation précise du contrat (pré/postconditions).
3. Créer les classes `Image`, `Video`, `AudioFile` et `LiveStream` qui implémentent uniquement les interfaces pertinentes.
4. Montrer qu'un service `MediaPlayer` peut dépendre uniquement de l'interface `Streamable` sans rien savoir du reste.
5. Discutez le lien entre ISP et le principe de moindre connaissance (Law of Demeter).

---

## 5. Dependency Inversion Principle (DIP)

### 5.1 Définition

> Les modules de haut niveau ne doivent pas dépendre des modules de bas niveau. Les deux doivent dépendre d'abstractions. Les abstractions ne doivent pas dépendre des détails. Les détails doivent dépendre des abstractions.

Ce principe, également de Robert C. Martin, est le socle de l'**injection de dépendances** et des architectures découplées. Il inverse le sens traditionnel des dépendances : au lieu que la couche métier appelle directement une couche infrastructure (base de données, e-mail, système de fichiers), elle définit un contrat (interface) que l'infrastructure respecte.

Le DIP est indispensable à la testabilité : il permet de substituer l'infrastructure réelle par des doublures de test (mocks, stubs, fakes).

### 5.2 Exemple PHP

**Violation du DIP**

La couche métier dépend directement de la classe concrète `MySqlUserRepository`.

```php
<?php

class MySqlUserRepository
{
    public function findByEmail(string $email): ?array
    {
        // Requête SQL directe...
        return null;
    }
}

class AuthService
{
    // Dépendance concrète : impossible de tester sans une base MySQL réelle.
    private MySqlUserRepository $repository;

    public function __construct()
    {
        $this->repository = new MySqlUserRepository();
    }

    public function login(string $email, string $password): bool
    {
        $user = $this->repository->findByEmail($email);
        return $user !== null && password_verify($password, $user['password_hash']);
    }
}
```

**Application du DIP**

```php
<?php

// Abstraction définie par la couche métier, pas par l'infrastructure.
interface UserRepositoryInterface
{
    public function findByEmail(string $email): ?array;
}

// Implémentation concrète dans la couche infrastructure.
class MySqlUserRepository implements UserRepositoryInterface
{
    public function __construct(private readonly PDO $pdo) {}

    public function findByEmail(string $email): ?array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE email = ?');
        $stmt->execute([$email]);
        return $stmt->fetch(PDO::FETCH_ASSOC) ?: null;
    }
}

// Implémentation alternative pour les tests.
class InMemoryUserRepository implements UserRepositoryInterface
{
    private array $users = [];

    public function add(array $user): void
    {
        $this->users[$user['email']] = $user;
    }

    public function findByEmail(string $email): ?array
    {
        return $this->users[$email] ?? null;
    }
}

// La couche métier ne dépend que de l'abstraction.
class AuthService
{
    public function __construct(
        private readonly UserRepositoryInterface $repository
    ) {}

    public function login(string $email, string $password): bool
    {
        $user = $this->repository->findByEmail($email);
        return $user !== null && password_verify($password, $user['password_hash']);
    }
}

// Composition à la racine de l'application (Composition Root) :
// $service = new AuthService(new MySqlUserRepository($pdo));
// En test :
// $repo = new InMemoryUserRepository();
// $repo->add(['email' => 'test@example.com', 'password_hash' => password_hash('secret', PASSWORD_BCRYPT)]);
// $service = new AuthService($repo);
```

### 5.3 Exemple Python

**Violation du DIP**

```python
import smtplib
from email.message import EmailMessage


class EmailNotifier:
    """Infrastructure concrète."""
    def send(self, to: str, subject: str, body: str) -> None:
        msg = EmailMessage()
        msg["To"] = to
        msg["Subject"] = subject
        msg.set_content(body)
        with smtplib.SMTP("localhost") as smtp:
            smtp.send_message(msg)


class OrderService:
    def __init__(self):
        # Instanciation directe : couplage fort à SMTP.
        self.notifier = EmailNotifier()

    def place_order(self, user_email: str, product: str) -> None:
        # ... logique métier ...
        self.notifier.send(user_email, "Commande confirmée", f"Votre commande de {product} est enregistrée.")
```

**Application du DIP**

```python
from abc import ABC, abstractmethod
import smtplib
from email.message import EmailMessage


# Abstraction définie par la couche métier.
class NotificationPort(ABC):
    @abstractmethod
    def notify(self, recipient: str, subject: str, message: str) -> None:
        ...


# Adaptateur SMTP (couche infrastructure).
class SmtpEmailAdapter(NotificationPort):
    def __init__(self, host: str = "localhost"):
        self.host = host

    def notify(self, recipient: str, subject: str, message: str) -> None:
        msg = EmailMessage()
        msg["To"] = recipient
        msg["Subject"] = subject
        msg.set_content(message)
        with smtplib.SMTP(self.host) as smtp:
            smtp.send_message(msg)


# Adaptateur SMS fictif — ajout sans modifier OrderService.
class SmsAdapter(NotificationPort):
    def notify(self, recipient: str, subject: str, message: str) -> None:
        print(f"[SMS -> {recipient}] {subject}: {message}")


# Doublure pour les tests.
class FakeNotifier(NotificationPort):
    def __init__(self):
        self.sent: list[dict] = []

    def notify(self, recipient: str, subject: str, message: str) -> None:
        self.sent.append({"to": recipient, "subject": subject, "message": message})


# Couche métier : dépend uniquement de l'abstraction.
class OrderService:
    def __init__(self, notifier: NotificationPort):
        self.notifier = notifier

    def place_order(self, user_email: str, product: str) -> None:
        # ... logique métier ...
        self.notifier.notify(
            user_email,
            "Commande confirmée",
            f"Votre commande de {product} est enregistrée."
        )


# Test unitaire sans dépendance à SMTP :
# fake = FakeNotifier()
# service = OrderService(fake)
# service.place_order("user@example.com", "Laptop")
# assert fake.sent[0]["to"] == "user@example.com"
```

### 5.4 Exercices

**Exercice 1 — Niveau débutant**

Refactorisez le code PHP suivant pour appliquer le DIP :

```php
<?php
class InvoiceService
{
    public function generate(int $orderId): string
    {
        $db = new PDO('mysql:host=localhost;dbname=shop', 'root', '');
        $stmt = $db->prepare('SELECT * FROM orders WHERE id = ?');
        $stmt->execute([$orderId]);
        $order = $stmt->fetch(PDO::FETCH_ASSOC);

        $pdf = new \TCPDF();
        $pdf->AddPage();
        $pdf->writeHTML("<h1>Facture #{$order['id']}</h1><p>Total : {$order['total']} €</p>");
        return $pdf->Output('', 'S');
    }
}
```

- Identifiez les deux dépendances concrètes.
- Créez une interface pour chacune.
- Réécrivez `InvoiceService` en dépendant uniquement des abstractions.
- Ecrivez un test unitaire (PHPUnit ou pseudo-code) utilisant des stubs.

---

**Exercice 2 — Niveau intermédiaire**

Concevez en Python un système de journalisation (logging) respectant le DIP :
- La couche métier (ex : `PaymentService`) doit pouvoir enregistrer des événements sans connaître la destination des logs.
- Les destinations possibles sont : console, fichier texte, base de données, service distant (HTTP).
- Implémentez trois adaptateurs de log et un `CompositeLogger` qui délègue à plusieurs destinations simultanément.
- Écrivez un test unitaire de `PaymentService` avec un `InMemoryLogger`.

---

**Exercice 3 — Niveau avancé**

Vous construisez en PHP une application de traitement de données financières. La chaîne de traitement est :

`Lecteur de source` → `Transformateur de données` → `Validateur` → `Ecrivain de destination`

Les sources possibles : CSV, API REST, base SQL.  
Les destinations possibles : base SQL, fichier JSON, file de messages (RabbitMQ).

1. Définissez les interfaces `DataSourceInterface`, `DataTransformerInterface`, `DataValidatorInterface` et `DataWriterInterface`.
2. Implémentez deux sources et deux destinations concrètes.
3. Implémentez un `DataPipeline` qui orchestre les quatre étapes en dépendant uniquement des abstractions.
4. Démontrez, via des tests unitaires avec doublures, que le pipeline fonctionne correctement sans aucune infrastructure réelle.
5. Discutez les limites du DIP : coût de l'abstraction, sur-ingénierie, et critères pour décider quand l'appliquer.

---

## Synthèse : relations entre les cinq principes

Les principes SOLID ne sont pas indépendants ; ils se renforcent mutuellement :

- Le **SRP** délimite les classes, ce qui rend les interfaces de l'**ISP** naturellement cohésives.
- L'**OCP** s'appuie sur des abstractions que le **DIP** impose de définir au bon niveau.
- Le **LSP** est la condition nécessaire pour que le polymorphisme utilisé par l'**OCP** soit correct.
- L'**ISP** prévient les couplages inutiles que le **DIP** cherche à éliminer.

Un moyen mnémotechnique utile :

> SRP dit *quoi* séparer.  
> OCP dit *comment* étendre sans casser.  
> LSP dit *quand* l'héritage est sûr.  
> ISP dit *combien* d'interfaces créer.  
> DIP dit *de qui* dépendre.

Ces principes ne garantissent pas à eux seuls un bon design, mais leur violation systématique est un indicateur fiable de code fragile, difficile à tester et à faire évoluer.

---

## Bibliographie

### Ouvrages fondamentaux

- **Martin, R. C.** (2002). *Agile Software Development, Principles, Patterns, and Practices*. Prentice Hall.  
  L'ouvrage de référence qui formalise les principes SOLID dans le contexte du développement agile.

- **Martin, R. C.** (2008). *Clean Code: A Handbook of Agile Software Craftsmanship*. Prentice Hall.  
  Complément indispensable : nommage, fonctions, structure du code — le SOLID au niveau microscopique.

- **Martin, R. C.** (2017). *Clean Architecture: A Craftsman's Guide to Software Structure and Design*. Prentice Hall.  
  Prolongement des principes SOLID vers l'architecture hexagonale et les systèmes en couches.

- **Gamma, E., Helm, R., Johnson, R., & Vlissides, J.** (1994). *Design Patterns: Elements of Reusable Object-Oriented Software*. Addison-Wesley.  
  Le « Gang of Four » : les patterns classiques dont plusieurs (Strategy, Decorator, Observer) sont des applications directes des principes SOLID.

- **Fowler, M.** (2018). *Refactoring: Improving the Design of Existing Code* (2e éd.). Addison-Wesley.  
  Le catalogue de référence des refactorisations ; indispensable pour appliquer SRP et OCP dans du code legacy.

### Ouvrages complémentaires

- **Evans, E.** (2003). *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Addison-Wesley.  
  Applique les principes SOLID à grande échelle, notamment SRP et DIP, via les Bounded Contexts et les Repository.

- **Vernon, V.** (2013). *Implementing Domain-Driven Design*. Addison-Wesley.  
  Guide pratique du DDD ; très utile pour ancrer le DIP et l'ISP dans des exemples métier concrets.

- **Feathers, M.** (2004). *Working Effectively with Legacy Code*. Prentice Hall.  
  Techniques pour introduire les principes SOLID dans une base de code existante sans tout réécrire.

- **Freeman, S. & Pryce, N.** (2009). *Growing Object-Oriented Software, Guided by Tests*. Addison-Wesley.  
  Montre comment TDD et SOLID se renforcent mutuellement dans la conception émergente.

### Articles et ressources en ligne

- **Liskov, B. H. & Wing, J.** (1994). A behavioral notion of subtyping. *ACM Transactions on Programming Languages and Systems*, 16(6), 1811–1841.  
  L'article original formalisant le Liskov Substitution Principle.

- **Martin, R. C.** (2000). Design principles and design patterns. *ObjectMentor*.  
  Le document original (non révisé) dans lequel les cinq principes sont présentés ensemble pour la première fois. Disponible sur le site de l'auteur.

- **Martin, R. C.** Blog : *The Clean Code Blog* — https://blog.cleancoder.com  
  Articles de référence sur les principes SOLID, l'architecture et le craftsmanship.

- **Fowler, M.** Blog : *MartinFowler.com* — https://martinfowler.com  
  Catalogue des patterns d'entreprise, articles sur la dette technique et le refactoring.
