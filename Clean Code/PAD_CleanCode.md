# Jour 1 — Matin : Du code propre à la maîtrise du refactoring

## Avant-propos : pourquoi les patterns et SOLID ne suffisent pas

Les design patterns et les principes SOLID constituent un vocabulaire commun
indispensable. Ils répondent à la question *comment structurer le code* à un niveau
macro. Mais ils ne répondent pas à une question tout aussi importante : *comment
écrire une ligne de code, une fonction, un nom de variable de façon à ce qu'un
humain comprenne immédiatement ce que fait ce code*.

Un programme peut parfaitement respecter SOLID et rester illisible : une classe
ne faisant qu'une seule chose mais dont les noms de méthodes sont ésotériques,
des abstractions impeccables mais des fonctions de trente paramètres, un beau
pattern Strategy composé de variables nommées `tmp`, `data` et `x`.

Robert C. Martin formule ce constat dans *Clean Code* (2008) :

> « La proportion de temps passé à lire du code par rapport au temps passé
> à en écrire est largement supérieure à 10 pour 1. »

Si lire du code occupe 90 % du temps d'un développeur, la lisibilité n'est pas
un luxe esthétique : c'est un levier de productivité direct. Ce cours complète
donc les acquis en patterns et SOLID par les pratiques qui agissent au niveau
le plus fin : le code lui-même.

---

## Session 1 — Clean Code & code smells avancés

### 1.1 Nommage expressif

Le nommage est la première forme de documentation. Un bon nom rend un commentaire
inutile ; un mauvais nom force à lire l'implémentation pour comprendre l'intention.

#### Principes fondamentaux

**Révéler l'intention** : le nom doit répondre à trois questions sans qu'on ait
besoin de lire le corps — pourquoi cette variable/fonction/classe existe, ce qu'elle
fait, comment elle s'utilise.

```python
# Mauvais : que signifie d ?
d = 86400

# Bon : l'intention est immédiate
SECONDS_PER_DAY = 86_400
```

```php
<?php
// Mauvais
function chk(array $u): bool
{
    return $u['a'] > 0 && isset($u['e']);
}

// Bon
function isEligibleForPromotion(array $user): bool
{
    return $user['age'] > 0 && isset($user['email']);
}
```

**Eviter la désinformation** : ne pas utiliser des noms qui évoquent une structure
différente de la réalité.

```python
# Mauvais : accountList n'est pas une list Python
account_list = {"alice": 100, "bob": 200}

# Bon
accounts_by_name = {"alice": 100, "bob": 200}
```

**Noms prononçables** : si l'on ne peut pas lire un nom à voix haute, la
communication en équipe devient difficile.

```php
<?php
// Mauvais
class DtaRcrd102 { public string $gnmdhms; }

// Bon
class CustomerRecord { public string $generationTimestamp; }
```

**Noms cherchables** : une constante nommée `MAX_CLASSES_PER_STUDENT` est
trouvable dans tout l'éditeur ; le chiffre `7` seul ne l'est pas.

**Un mot par concept** : `fetch`, `get`, `retrieve` ne doivent pas coexister
dans la même base de code pour désigner la même opération. Choisir l'un et
s'y tenir.

**Nommer selon le domaine métier** : une méthode `calculateRiskScore` dit plus
qu'une méthode `process`. Le code doit parler le même langage que les
spécifications et les discussions avec les experts métier — c'est l'essence de
l'Ubiquitous Language que vous découvrirez cet après-midi avec le DDD.

#### Conventions par type d'entité

| Entité | Convention | Exemples |
|---|---|---|
| Classe | Nom (substantif), PascalCase | `InvoiceGenerator`, `UserRepository` |
| Méthode | Verbe + complément | `generateMonthlyInvoice()`, `findByEmail()` |
| Booléen | Préfixe `is`, `has`, `can`, `should` | `isActive`, `hasPermission` |
| Constante | SCREAMING_SNAKE_CASE | `MAX_RETRY_ATTEMPTS` |
| Paramètre | Descriptif, pas abbrévié | `recipientEmail`, pas `e` |

---

### 1.2 Fonctions courtes et abstraction par niveau

#### La règle de l'abstraction uniforme

Une fonction ne devrait faire qu'une chose, et tous ses appels internes devraient
se situer au même niveau d'abstraction. Mélanger des opérations de haut niveau
(« envoyer la notification ») avec des détails de bas niveau (manipuler des
octets, construire une chaîne SQL) dans la même fonction est l'une des sources
les plus fréquentes de code difficile à lire.

```python
# Mauvais : mélange de niveaux d'abstraction
def process_order(order_id: int) -> None:
    # Bas niveau : SQL brut
    conn = sqlite3.connect("shop.db")
    row  = conn.execute("SELECT * FROM orders WHERE id = ?", (order_id,)).fetchone()

    # Haut niveau : règle métier
    if row["status"] == "pending" and row["total"] > 0:
        # Bas niveau : construction manuelle d'un payload JSON
        payload = '{"order_id":' + str(order_id) + ',"event":"confirmed"}'
        urllib.request.urlopen("https://notify.example.com", payload.encode())

        # Bas niveau : mise à jour SQL
        conn.execute("UPDATE orders SET status='confirmed' WHERE id=?", (order_id,))
        conn.commit()
```

```python
# Bon : chaque fonction est à un seul niveau d'abstraction
def process_order(order_id: int) -> None:
    order = fetch_order(order_id)
    if order.is_confirmable():
        confirm_order(order)
        notify_confirmation(order)

def fetch_order(order_id: int) -> Order:
    ...  # bas niveau : SQL

def confirm_order(order: Order) -> None:
    ...  # bas niveau : UPDATE SQL

def notify_confirmation(order: Order) -> None:
    ...  # bas niveau : appel HTTP
```

La version lisible se lit comme un texte : *on récupère la commande, si elle est
confirmable on la confirme et on notifie*. Aucun détail d'infrastructure ne pollue
cette lecture.

#### Nombre de paramètres

Les fonctions à un paramètre (monadiques) sont idéales. À deux (dyadiques), c'est
acceptable. Au-delà de trois, il faut sérieusement envisager un objet de paramètres.

```php
<?php
// Mauvais : 7 paramètres, ordre arbitraire, difficile à appeler correctement
function createUser(
    string $firstName, string $lastName, string $email,
    string $role, bool $isActive, string $country, int $age
): User { ... }

// Bon : objet de paramètres nommé
final class UserCreationRequest
{
    public function __construct(
        public readonly string $firstName,
        public readonly string $lastName,
        public readonly string $email,
        public readonly string $role     = 'user',
        public readonly bool   $isActive = true,
        public readonly string $country  = 'FR',
        public readonly int    $age      = 0,
    ) {}
}

function createUser(UserCreationRequest $request): User { ... }
```

#### Eviter les effets de bord cachés

Une fonction dont le nom dit `checkPassword` ne devrait pas initialiser une session,
envoyer un e-mail ou modifier une base de données. Les effets de bord non annoncés
sont une source majeure de bugs difficiles à tracer.

```php
<?php
// Mauvais : checkPassword fait bien plus que vérifier
public function checkPassword(string $email, string $password): bool
{
    $user = $this->repository->findByEmail($email);
    if ($user && password_verify($password, $user->passwordHash)) {
        $this->session->initialize($user);  // effet de bord caché
        $this->logger->info("Login réussi pour $email");  // idem
        return true;
    }
    return false;
}

// Bon : séparation vérification / effet de bord
public function verifyCredentials(string $email, string $password): bool
{
    $user = $this->repository->findByEmail($email);
    return $user !== null && password_verify($password, $user->passwordHash);
}

public function login(string $email, string $password): void
{
    if (!$this->verifyCredentials($email, $password)) {
        throw new AuthenticationException("Identifiants invalides.");
    }
    $user = $this->repository->findByEmail($email);
    $this->session->initialize($user);
    $this->logger->info("Login réussi pour $email");
}
```

---

### 1.3 Loi de Déméter et principe du moindre étonnement

#### Loi de Déméter (LoD)

La Loi de Déméter — parfois appelée « principe du moindre couplage » ou règle
« ne parle qu'à tes amis directs » — a été formulée en 1987 par Karl Lieberherr
et Ian Holland au Northeastern University.

Un objet `O` appelant une méthode `m` ne devrait interagir qu'avec :
- lui-même ;
- ses attributs directs ;
- les objets passés en paramètre de `m` ;
- les objets créés dans le corps de `m` ;
- les variables globales accessibles à `O`.

Il ne devrait pas appeler des méthodes sur les objets *retournés* par d'autres
méthodes — c'est le « train wreck » (accident ferroviaire).

```python
# Mauvais : train wreck — on traverse plusieurs objets
# Si l'un d'eux change d'API, toutes ces lignes sont à modifier
discount = order.get_customer().get_loyalty_account().get_discount_rate()

# Bon : on délègue la responsabilité à l'objet concerné
discount = order.get_applicable_discount()
# Order interroge Customer, qui interroge LoyaltyAccount — en interne.
```

```php
<?php
// Mauvais
$zip = $this->order->getShippingAddress()->getCity()->getZipCode();

// Bon
$zip = $this->order->getShippingZipCode();
// Order encapsule la traversée
```

En PHP, les chaînes de méthodes fluentes (builder, query builder) sont une
exception documentée à la LoD : `$query->select('*')->from('users')->where('id', 1)`
est un DSL intentionnel, pas une traversée accidentelle d'objets.

#### Principe du moindre étonnement (POLA)

Le POLA (*Principle of Least Astonishment*) stipule qu'un composant doit se
comporter de la façon dont ses utilisateurs s'attendent qu'il se comporte.
En d'autres termes : ne pas surprendre.

```python
# Mauvais : get_user() produit un effet de bord inattendu
def get_user(user_id: int) -> User:
    user = self.db.find(user_id)
    self.audit_log.record_access(user_id)  # surprise : un getter écrit dans un log
    return user

# Bon : séparer la consultation et la journalisation
def get_user(user_id: int) -> User:
    return self.db.find(user_id)

def get_user_with_audit(user_id: int) -> User:
    user = self.get_user(user_id)
    self.audit_log.record_access(user_id)
    return user
```

Autres formes de violation du POLA :
- une méthode `save()` qui retourne un booléen dans certains cas et lève une
  exception dans d'autres ;
- un constructeur qui effectue des appels réseau ou lit des fichiers ;
- une méthode `isEmpty()` qui modifie l'état interne de l'objet.

---

### 1.4 Dette technique — définition et mesure

#### Définition et origines

Ward Cunningham a introduit la métaphore de la « dette technique » en 1992 :
prendre un raccourci dans le code aujourd'hui, c'est contracter une dette dont
les intérêts se paient sous forme de temps de développement supplémentaire dans
le futur. Comme une dette financière, elle peut être gérée, remboursée
progressivement — ou laissée s'accumuler jusqu'à l'asphyxie.

La dette technique n'est pas intrinsèquement mauvaise. Elle peut être
*délibérée* et *prudente* : livrer rapidement une première version pour valider
une hypothèse métier, avec l'intention explicite de rembourser plus tard. Le
problème survient quand elle est *inconsciente* ou *irresponsable*.

#### Le quadrant de Fowler (2009)

Martin Fowler propose de classifier la dette selon deux axes :

```
                 DELIBEREE
                     |
  IMPRUDENTE         |         PRUDENTE
  "On n'a pas        |  "On livre maintenant
   le temps de       |   et on refactore
   bien faire"       |   ensuite"
                     |
  ─────────────────────────────────────────
                     |
  IMPRUDENTE         |         PRUDENTE
  "C'est quoi        |  "On savait qu'il
   un pattern ?"     |   fallait faire autrement,
                     |   mais on ne savait pas
                     |   comment"
                INCONSCIENTE
```

La dette *délibérée et prudente* est la seule forme acceptable. Elle suppose :
- une décision consciente et documentée ;
- un plan de remboursement (backlog, Architecture Decision Record) ;
- une communication transparente avec l'équipe et les parties prenantes.

#### Mesure de la dette

Plusieurs outils permettent d'objectiver la dette dans une base de code :

**Métriques de code statique**

| Métrique | Ce qu'elle mesure | Seuil d'alerte courant |
|---|---|---|
| Complexité cyclomatique | Nombre de chemins indépendants dans une fonction | > 10 |
| Longueur de fonction | Nombre de lignes | > 20–30 |
| Profondeur d'imbrication | Niveaux de if/for/while emboîtés | > 3 |
| Couplage afférent (Ca) | Nombre de classes qui dépendent de X | > 20 |
| Couplage efférent (Ce) | Nombre de classes dont X dépend | > 10 |
| LCOM (Lack of Cohesion) | Classes dont les méthodes ne partagent pas d'attributs | > 0.8 |

**Outils**

- Python : `radon`, `pylint`, `flake8`, `mypy`
- PHP : `phpstan`, `phpmd` (PHP Mess Detector), `phploc`, `psalm`
- Transversaux : SonarQube, Code Climate

**La règle du boy scout**

Robert C. Martin reformule une règle simple : *laissez le code dans un meilleur
état que vous ne l'avez trouvé*. Chaque passage dans un fichier est l'occasion
de rembourser un peu de dette : renommer une variable, extraire une fonction,
ajouter un test. Ces micro-remboursements, accumulés, évitent le grand refactoring
douloureux.

---

### 1.5 Code smells avancés

Les « code smells » (ou mauvaises odeurs) sont des signaux dans le code qui
suggèrent un problème de conception potentiel. Ce ne sont pas des bugs — le code
peut fonctionner correctement tout en dégageant une mauvaise odeur. Voici les
smells les plus importants au-delà des classiques (fonction trop longue, classe
trop grande) que vous connaissez déjà.

---

#### Feature Envy (Envie de fonctionnalité)

**Définition :** une méthode qui semble plus intéressée par les données d'une autre
classe que par celles de la sienne propre. Elle appelle de nombreux accesseurs sur
un objet externe plutôt que de travailler sur `self`.

**Signal :** un enchaînement de `getX()`, `getY()`, `getZ()` sur le même objet
externe dans une seule méthode.

```python
# Mauvais : OrderPrinter est "jalouse" de Order
class OrderPrinter:
    def print_summary(self, order: Order) -> str:
        # Cette méthode ne fait qu'interroger Order — elle devrait y être
        customer = order.get_customer()
        lines    = order.get_lines()
        discount = order.get_discount()
        total    = order.get_subtotal() - discount

        return (
            f"Client : {customer.get_full_name()}\n"
            f"Lignes : {len(lines)}\n"
            f"Remise : {discount:.2f} €\n"
            f"Total  : {total:.2f} €"
        )

# Bon : la logique de résumé appartient à Order
class Order:
    def summary(self) -> str:
        total = self.subtotal - self.discount
        return (
            f"Client : {self.customer.full_name}\n"
            f"Lignes : {len(self.lines)}\n"
            f"Remise : {self.discount:.2f} €\n"
            f"Total  : {total:.2f} €"
        )
```

**Contre-exemple légitime :** les objets de type Strategy ou Visitor ont vocation
à opérer sur des données externes — ce n'est pas du Feature Envy car c'est
délibéré et structurel.

---

#### Shotgun Surgery (Chirurgie au fusil de chasse)

**Définition :** l'inverse du Feature Envy. Un seul changement conceptuel
(modifier un comportement, changer une règle) oblige à modifier de nombreux
fichiers dispersés dans la base de code.

**Signal :** toute modification d'une règle métier ou d'un format nécessite
de retrouver et de toucher cinq fichiers ou plus.

```php
<?php
// Symptôme : la règle "TVA à 20 %" est codée en dur dans plusieurs classes

class InvoiceCalculator {
    public function compute(float $subtotal): float {
        return $subtotal * 1.20;  // TVA ici
    }
}

class QuoteGenerator {
    public function estimate(float $cost): float {
        return $cost * 1.20;  // TVA là
    }
}

class CartSummary {
    public function total(array $items): float {
        return array_sum($items) * 1.20;  // TVA encore
    }
}

// Correction : centraliser la règle
final class TaxCalculator
{
    private const VAT_RATE = 0.20;

    public function applyVat(float $amount): float
    {
        return $amount * (1 + self::VAT_RATE);
    }
}
// InvoiceCalculator, QuoteGenerator et CartSummary injectent TaxCalculator.
// Si le taux change, on ne modifie qu'un seul endroit.
```

**Lien avec SOLID :** le Shotgun Surgery est souvent la conséquence d'une
violation du DIP (dépendances directes à des valeurs concrètes) et du SRP
(la responsabilité de calculer la TVA est dispersée).

---

#### Divergent Change (Changement divergent)

**Définition :** l'opposé du Shotgun Surgery. Une seule classe change pour des
raisons multiples et non liées — elle subit une « divergence » à chaque fois
qu'une règle d'un domaine différent évolue.

**Signal :** en cherchant l'historique Git de la classe, on voit des commits
provenant de contextes totalement différents (modification du format CSV, puis
d'une règle de validation, puis d'un calcul de remise).

```python
# Mauvais : ReportService change pour trois raisons distinctes
class ReportService:
    # Raison 1 : modification du format de sortie (CSV, JSON, XML)
    def export(self, data: list, format: str) -> str: ...

    # Raison 2 : modification des règles de filtrage métier
    def filter_active_accounts(self, accounts: list) -> list: ...

    # Raison 3 : modification du calcul des agrégats statistiques
    def compute_monthly_totals(self, transactions: list) -> dict: ...

# Bon : trois classes, chacune avec une seule raison de changer
class ReportExporter:
    def export(self, data: list, format: str) -> str: ...

class AccountFilter:
    def filter_active(self, accounts: list) -> list: ...

class TransactionAggregator:
    def monthly_totals(self, transactions: list) -> dict: ...
```

**Divergent Change vs Shotgun Surgery :** ils sont complémentaires et souvent
présents ensemble. Le Divergent Change dit « *cette* classe change trop souvent
et pour trop de raisons » ; le Shotgun Surgery dit « *ce* changement touche trop
de classes ». La correction dans les deux cas passe par un meilleur respect du SRP.

---

#### Autres smells notables

**Primitive Obsession :** utiliser des types primitifs (`string`, `int`, `float`)
là où un objet valeur serait plus expressif et plus sûr.

```php
<?php
// Mauvais : email est une chaîne, rien n'empêche "pas_un_email"
function sendInvoice(string $email, float $amount): void { ... }

// Bon : Email garantit la validité à la construction
final class Email {
    public function __construct(public readonly string $value) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Email invalide : $value");
        }
    }
}
final class Money {
    public function __construct(
        public readonly int    $amountCents,
        public readonly string $currency,
    ) {
        if ($amountCents < 0) throw new \InvalidArgumentException("Montant négatif.");
    }
}
function sendInvoice(Email $recipient, Money $amount): void { ... }
```

**Data Clumps :** un groupe de variables qui apparaissent toujours ensemble (nom,
prénom, email) mérite d'être regroupé dans un objet dédié.

**Middle Man :** une classe qui ne fait que déléguer à une autre sans aucune valeur
ajoutée. Supprimer l'intermédiaire ou lui donner une vraie responsabilité.

**Long Parameter List :** voir la section 1.2 — au-delà de trois paramètres,
envisager un objet de paramètres.

**Comments (code mort commenté) :** du code commenté s'accumule et pollue.
Le contrôle de version (Git) joue ce rôle : supprimer le code mort, le retrouver
dans l'historique si besoin.

---

## Session 2 — TD pratique : Refactoring d'une base de code existante

### 2.1 Introduction : qu'est-ce que le refactoring ?

Martin Fowler définit le refactoring comme :

> « Une technique disciplinée de restructuration du code existant qui consiste à
> modifier sa structure interne sans modifier son comportement observable. »

Deux mots clés : **disciplinée** (outillée, pas à l'instinct) et **sans modifier
le comportement observable** (les tests passent avant et après). Sans cette
deuxième contrainte, on fait de la réécriture, pas du refactoring.

Le refactoring ne corrige pas des bugs (même si des bugs peuvent apparaître
à la surface pendant le travail) et n'ajoute pas de fonctionnalités. C'est
une activité de *préparation* : on assainit le code pour que la prochaine
fonctionnalité soit plus facile à ajouter.

---

### 2.2 Le catalogue de Fowler — les refactorisations fondamentales

Voici les six refactorisations les plus fréquemment utilisées, toutes disponibles
sous forme d'actions automatisées dans PHPStorm, IntelliJ et VS Code.

#### Extract Method / Extract Function

**Quand :** une section de code forme une unité sémantique identifiable, ou un
commentaire décrit ce que fait un bloc.

```python
# Avant
def print_receipt(order):
    # Calcul du total
    subtotal = sum(line.price * line.qty for line in order.lines)
    discount = subtotal * 0.10 if order.is_member else 0
    total = subtotal - discount

    # Impression
    print(f"Sous-total : {subtotal:.2f} €")
    print(f"Remise     : {discount:.2f} €")
    print(f"Total      : {total:.2f} €")

# Après : Extract Function sur le bloc de calcul
def calculate_total(order) -> tuple[float, float, float]:
    subtotal = sum(line.price * line.qty for line in order.lines)
    discount = subtotal * 0.10 if order.is_member else 0
    return subtotal, discount, subtotal - discount

def print_receipt(order):
    subtotal, discount, total = calculate_total(order)
    print(f"Sous-total : {subtotal:.2f} €")
    print(f"Remise     : {discount:.2f} €")
    print(f"Total      : {total:.2f} €")
```

#### Rename

La refactorisation la plus simple et la plus impactante. Disponible sur toute
l'étendue du projet dans les IDE modernes.

**Règle :** si une variable porte un commentaire explicatif, le commentaire devrait
devenir le nom.

```php
<?php
// Avant
$d = 7;  // nombre de jours dans la période d'essai
if ($daysSinceSignup > $d) { ... }

// Après : Rename
$trialPeriodDays = 7;
if ($daysSinceSignup > $trialPeriodDays) { ... }
```

#### Move Method / Move Function

**Quand :** une méthode utilise davantage les données d'une autre classe que
de la sienne (Feature Envy).

```php
<?php
// Avant : la méthode est dans Account mais utilise surtout Customer
class Account {
    public function getCustomerFullName(): string {
        return $this->customer->getFirstName() . ' ' . $this->customer->getLastName();
    }
}

// Après : la méthode est déplacée dans Customer (via Move Method)
class Customer {
    public function getFullName(): string {
        return $this->firstName . ' ' . $this->lastName;
    }
}
// Account délègue : $this->customer->getFullName()
```

#### Inline Variable / Inline Method

L'inverse d'Extract : supprimer une indirection qui n'apporte pas de lisibilité.

```python
# Avant : variable intermédiaire inutile
base_price = order.base_price()
if base_price > 1000:
    ...

# Après
if order.base_price() > 1000:
    ...
```

#### Introduce Parameter Object

Voir section 1.2 — regrouper un ensemble de paramètres qui voyagent toujours
ensemble dans une classe dédiée.

#### Replace Conditional with Polymorphism

**Quand :** un `switch` ou un `if/elif` cascade sur un type ou un statut, et
doit être étendu régulièrement.

```python
# Avant
def calculate_pay(employee):
    if employee.type == "engineer":
        return engineer_pay(employee)
    elif employee.type == "manager":
        return manager_pay(employee)
    elif employee.type == "salesperson":
        return salesperson_pay(employee)

# Après : polymorphisme (OCP)
class Employee(ABC):
    @abstractmethod
    def calculate_pay(self) -> float: ...

class Engineer(Employee): ...
class Manager(Employee): ...
class Salesperson(Employee): ...
```

---

### 2.3 Les Mikado Maps — planifier un refactoring complexe

Le refactoring d'une base de code legacy présente un défi particulier : chaque
modification peut en révéler plusieurs autres nécessaires, créant un effet
boule-de-neige qui aboutit à un chantier interminable.

La **Mikado Method** (Ola Ellneby & Henrik Kniberg, 2010) propose une approche
systématique pour naviguer ce problème.

#### Le principe du jeu de Mikado

Dans le jeu de Mikado, retirer un bâtonnet trop vite fait tomber les autres.
La méthode informatique qui porte son nom procède de même : avant de modifier
le code, on identifie les prérequis de cette modification, puis les prérequis
de ces prérequis, formant un arbre de dépendances. On commence par les feuilles
(sans prérequis) pour remonter vers la racine (l'objectif initial).

#### Algorithme en cinq étapes

```
1. Définir l'objectif (la racine) : "Remplacer SQLite par PostgreSQL dans UserRepository"

2. Essayer de l'atteindre directement :
   - Modifier le code
   - Compiler / exécuter les tests

3. Si les tests cassent :
   - Revert immédiat (git checkout -- .)
   - Ajouter un nœud enfant dans le graphe pour chaque prérequis découvert

4. Répéter depuis le bas du graphe (feuilles)

5. Quand une feuille est atteinte sans casse : valider et monter d'un niveau
```

#### Exemple de graphe Mikado

Objectif : *Introduire un cache Redis dans `ProductRepository`*

```
[Objectif] Introduire cache Redis dans ProductRepository
    ├── [Prérequis 1] ProductRepository dépend d'une interface (DIP)
    │       └── [Prérequis 1.1] Extraire ProductRepositoryInterface
    │               └── [Prérequis 1.1.1] Les tests utilisent l'interface, pas la classe concrète
    │
    ├── [Prérequis 2] CachedProductRepository implémente l'interface
    │       └── [Prérequis 2.1] RedisClient est injectable (pas instancié dans le constructeur)
    │
    └── [Prérequis 3] La configuration choisit entre SqlProductRepository et CachedProductRepository
            └── [Prérequis 3.1] Un Composition Root est défini
```

On commence par Prérequis 1.1.1 (une feuille) : écrire les tests sur l'interface.
Puis 1.1 : extraire l'interface. Puis 1 : refactoriser le repository. Et ainsi de
suite jusqu'à l'objectif.

**Bénéfice clé :** à tout moment, le code en cours est dans un état valide (les
tests passent ou on a revert). On n'est jamais dans un état intermédiaire bloquant.

---

### 2.4 Refactorer ou réécrire ?

La question « quand refactorer plutôt que réécrire » revient régulièrement. Il
n'existe pas de réponse universelle, mais des critères orientent la décision.

#### Arguments pour le refactoring progressif

- **Préservation du savoir tacite :** du code legacy, même mal écrit, encode
  souvent des règles métier que personne ne sait reformuler. La réécriture les
  efface. Joel Spolsky (2000) appelle cela *Things You Should Never Do*.
- **Continuité de service :** le refactoring incrémental permet de livrer des
  améliorations sans gel des fonctionnalités.
- **Risque maîtrisé :** chaque étape est couverte par les tests existants.

#### Arguments pour la réécriture

- Le code est tellement couplé qu'aucun test n'est possible sans infrastructure
  complète (c'est souvent le signe d'un couplage à l'infrastructure qui ne peut
  pas être extrait progressivement).
- Les exigences non fonctionnelles ont radicalement changé (ex. : passage d'un
  monolithe à un système distribué temps réel).
- Les fondations technologiques sont obsolètes et non mainteables (framework
  abandonné, version EOL sans support de sécurité).

#### Règle pratique

> Préférez toujours le refactoring progressif, sauf si vous pouvez démontrer
> que les tests d'intégration de l'existant couvrent le comportement observable
> et peuvent être réutilisés pour valider la réécriture.

Sans filet de tests, une réécriture est aussi risquée que le code qu'elle remplace.

---

### 2.5 Exercices pratiques

Ces exercices sont conçus pour être réalisés en binôme, avec une revue collective
à mi-parcours.

---

#### Exercice A — Identifier et nommer les smells

Analysez la classe Python suivante. Pour chaque problème identifié :
1. nommez le smell (parmi ceux vus en cours) ;
2. indiquez quelle refactorisation y remédierait ;
3. proposez un pseudo-code de la version corrigée.

```python
class Mgr:

    def __init__(self):
        import sqlite3
        self.c = sqlite3.connect("app.db")
        self.l = []
        self.x = 0

    def do(self, t, d, e, p, q):
        # check
        if t == "" or d == "" or e == "" or p <= 0 or q <= 0:
            return False

        # db
        cur = self.c.execute(
            "SELECT id, price, stock FROM products WHERE id = ?", (p,)
        )
        row = cur.fetchone()
        if not row:
            return False

        pid, price, stock = row

        if q > stock:
            return False

        tot = price * q
        if e.find("@") == -1 or e.find(".") == -1:
            return False

        disc = 0
        if q >= 10:
            disc = tot * 0.05
        if q >= 50:
            disc = tot * 0.10
        if q >= 100:
            disc = tot * 0.15
        tot = tot - disc

        self.c.execute(
            "INSERT INTO orders (title, description, email, product_id, quantity, total) "
            "VALUES (?, ?, ?, ?, ?, ?)",
            (t, d, e, pid, q, tot)
        )
        self.c.execute(
            "UPDATE products SET stock = stock - ? WHERE id = ?", (q, pid)
        )
        self.c.commit()

        import smtplib
        from email.message import EmailMessage
        msg = EmailMessage()
        msg["To"]      = e
        msg["From"]    = "shop@example.com"
        msg["Subject"] = "Commande confirmée"
        msg.set_content(f"Votre commande de {q} unités pour {tot:.2f}€ est confirmée.")
        try:
            with smtplib.SMTP("localhost") as s:
                s.send_message(msg)
        except Exception:
            pass

        self.l.append({"t": t, "tot": tot})
        self.x += 1
        return True

    def rep(self):
        print(f"Total commandes: {self.x}")
        for o in self.l:
            print(o["t"], o["tot"])
```

**Smells à identifier (liste non exhaustive pour guider) :**
- Nommage (variables, méthodes, classe)
- Responsabilités multiples
- Feature Envy
- Divergent Change
- Primitive Obsession
- Magic numbers
- Comments absents là où nécessaires, ou leur substitut : du code auto-documenté

---

#### Exercice B — Refactoring guidé d'une classe PHP

La classe suivante gère l'affichage d'un panier d'achat dans une application
e-commerce. Refactorisez-la en appliquant les principes vus en cours.

Contraintes :
- respecter le SRP (une classe, une raison de changer) ;
- éliminer tous les code smells identifiables ;
- les tests fournis ci-dessous doivent passer après refactoring ;
- ne pas modifier les signatures des tests.

```php
<?php
class Cart
{
    private array $i = [];  // items
    private ?string $pc = null;  // promo code
    private static array $codes = [
        'SUMMER' => 10,
        'VIP'    => 25,
        'STAFF'  => 50,
    ];

    public function add($pid, $name, $p, $q)
    {
        foreach ($this->i as &$item) {
            if ($item['id'] == $pid) {
                $item['q'] += $q;
                return;
            }
        }
        $this->i[] = ['id' => $pid, 'n' => $name, 'p' => $p, 'q' => $q];
    }

    public function setCode($c)
    {
        if (array_key_exists($c, self::$codes)) {
            $this->pc = $c;
        }
    }

    public function calc()
    {
        $s = 0;
        foreach ($this->i as $item) {
            $s += $item['p'] * $item['q'];
        }
        if ($this->pc !== null) {
            $d = self::$codes[$this->pc];
            $s = $s - ($s * $d / 100);
        }
        return $s;
    }

    public function html()
    {
        $o = '<table>';
        foreach ($this->i as $item) {
            $o .= '<tr><td>'.$item['n'].'</td>';
            $o .= '<td>'.$item['q'].'</td>';
            $o .= '<td>'.number_format($item['p'], 2).' €</td>';
            $o .= '<td>'.number_format($item['p'] * $item['q'], 2).' €</td>';
            $o .= '</tr>';
        }
        $o .= '</table>';
        $o .= '<p>Total : '.number_format($this->calc(), 2).' €</p>';
        if ($this->pc) {
            $o .= '<p>Code promo appliqué : '.$this->pc.' (-'.self::$codes[$this->pc].'%)</p>';
        }
        return $o;
    }

    public function count(): int { return count($this->i); }

    public function clear() { $this->i = []; $this->pc = null; }
}
```

**Tests de non-régression (ne pas modifier) :**

```php
<?php
use PHPUnit\Framework\TestCase;

final class CartTest extends TestCase
{
    public function testAddItemIncreasesCount(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 9.99, 2);
        $this->assertSame(1, $cart->count());
    }

    public function testAddSameProductMergesQuantity(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 9.99, 2);
        $cart->add(1, 'Widget', 9.99, 3);
        $this->assertSame(1, $cart->count());
    }

    public function testCalculatesTotalWithoutPromo(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 10.00, 3);
        $this->assertEqualsWithDelta(30.00, $cart->calc(), 0.001);
    }

    public function testAppliesPromoCodeDiscount(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 100.00, 1);
        $cart->setCode('SUMMER');  // 10 %
        $this->assertEqualsWithDelta(90.00, $cart->calc(), 0.001);
    }

    public function testIgnoresUnknownPromoCode(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 100.00, 1);
        $cart->setCode('INVALID');
        $this->assertEqualsWithDelta(100.00, $cart->calc(), 0.001);
    }

    public function testClearResetsCart(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 9.99, 1);
        $cart->clear();
        $this->assertSame(0, $cart->count());
    }
}
```

---

#### Exercice C — Construire une Mikado Map

Vous reprenez une application PHP legacy. La classe `UserController` contient
directement des requêtes PDO. Vous souhaitez atteindre l'objectif suivant :

**Objectif :** rendre `UserController` testable unitairement sans base de données.

Construisez la Mikado Map correspondante :
1. Listez toutes les modifications nécessaires.
2. Identifiez les dépendances entre ces modifications.
3. Dessinez le graphe (sur papier ou avec un outil de votre choix).
4. Numérotez les étapes dans l'ordre d'exécution (des feuilles vers la racine).

```php
<?php
class UserController
{
    public function show(int $id): string
    {
        $pdo  = new PDO('mysql:host=localhost;dbname=app', 'root', '');
        $stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$id]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$user) {
            http_response_code(404);
            return '<h1>Utilisateur introuvable</h1>';
        }
        return '<h1>' . htmlspecialchars($user['name']) . '</h1>';
    }

    public function index(): string
    {
        $pdo  = new PDO('mysql:host=localhost;dbname=app', 'root', '');
        $stmt = $pdo->query('SELECT id, name FROM users ORDER BY name');
        $rows = $stmt->fetchAll(PDO::FETCH_ASSOC);

        $html = '<ul>';
        foreach ($rows as $row) {
            $html .= '<li>' . htmlspecialchars($row['name']) . '</li>';
        }
        return $html . '</ul>';
    }
}
```

## Résumé

| Concept | Ce qu'il résout | Outil / Pattern associé |
|---|---|---|
| Nommage expressif | Code illisible, commentaires inutiles | Convention d'équipe, revue de code |
| Abstraction par niveau | Fonctions mélangées haut/bas niveau | Extract Method, SRP |
| Loi de Déméter | Couplage accidentel entre objets | Tell, don't ask |
| POLA | Comportements surprenants | Conception des API, revue |
| Dette technique | Accumulation de raccourcis non documentés | Quadrant Fowler, ADR, règle du boy scout |
| Feature Envy | Méthode mal placée | Move Method |
| Shotgun Surgery | Règle dispersée dans le code | Extract Class, DIP |
| Divergent Change | Classe qui change pour trop de raisons | Extract Class, SRP |
| Refactoring | Amélioration sans changement de comportement | Tests + IDE (extract, rename, move) |
| Mikado Method | Refactoring complexe sans blocage | Graphe de prérequis, revert systématique |

L'après-midi prolonge ces pratiques à l'échelle de l'architecture, avec le
Domain-Driven Design et l'Event Storming.

---

## Bibliographie

### Ouvrages de référence

**Martin, R. C.** (2008). *Clean Code: A Handbook of Agile Software Craftsmanship*. Prentice Hall.  
La source principale de cette matinée. Les chapitres 2 (nommage), 3 (fonctions), 4 (commentaires)
et 17 (smells et heuristiques) couvrent directement les sujets de la session 1. La lecture des
chapitres 2 et 3 est recommandée avant la formation.

**Fowler, M.** (2018). *Refactoring: Improving the Design of Existing Code* (2e éd.). Addison-Wesley.  
Le catalogue de référence absolu du refactoring. La première édition (1999) a introduit la
terminologie des code smells et le catalogue des refactorisations atomiques (Extract Method,
Move Method, Rename…). La deuxième édition reprend les exemples en JavaScript. Les chapitres 1
(exemple fil rouge), 3 (catalogue des smells) et 6 à 12 (catalogue des refactorisations) sont
les plus utiles pour cette matinée.

**Feathers, M.** (2004). *Working Effectively with Legacy Code*. Prentice Hall.  
Ouvrage incontournable sur le refactoring de code sans tests. Introduit les « seams »
(points d'entrée pour les tests) et des techniques pour rendre du code legacy testable
sans le réécrire. Complément direct de la session 2 (TD refactoring) et de la Mikado Method.

**Ellneby, O. & Kniberg, H.** (2014). *The Mikado Method*. Manning Publications.  
Présentation complète de la méthode Mikado, illustrée par des exemples réels de refactoring
complexe. Très court (180 pages), accessible à un développeur junior. Une version préliminaire
est disponible gratuitement sur le site de Manning.

### Articles et textes fondateurs

**Cunningham, W.** (1992). The WyCash Portfolio Management System.  
*OOPSLA '92 Experience Report*, ACM.  
Le texte dans lequel Ward Cunningham introduit pour la première fois la métaphore de la
dette technique. Court (une page), disponible en ligne sur le site de Cunningham.
À lire pour comprendre l'intention originale du concept, souvent dénaturé depuis.

**Lieberherr, K. & Holland, I.** (1989). Assuring good style for object-oriented programs.
*IEEE Software*, 6(5), 38–48.  
L'article de référence sur la Loi de Déméter. Formule la règle de façon rigoureuse
(« Only talk to your immediate friends ») et en donne les fondements théoriques.

**Fowler, M.** (2009). *TechnicalDebtQuadrant*. MartinFowler.com.  
Article de blog (environ 600 mots) qui introduit le quadrant de la dette technique :
délibérée/inconsciente × prudente/imprudente. Disponible à l'adresse
https://martinfowler.com/bliki/TechnicalDebtQuadrant.html

**Spolsky, J.** (2000). Things You Should Never Do, Part I. *Joel on Software*.  
Article de blog qui présente l'argument contre la réécriture complète d'une application
(*the single worst strategic mistake that any software company can make*). Utile comme
point de départ pour la discussion « refactorer ou réécrire » de la session 2.
Disponible à l'adresse https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/

### Pour aller plus loin

**Beck, K.** (1999). *Extreme Programming Explained: Embrace Change*. Addison-Wesley.  
Ouvrage qui formalise YAGNI (*You Aren't Gonna Need It*) et la règle du boy scout dans
le contexte de l'eXtreme Programming. Utile pour situer le refactoring dans un processus
de développement agile.

**Martin, R. C.** (2002). *Agile Software Development, Principles, Patterns, and Practices*.
Prentice Hall.  
Introduit les principes SOLID (abordés dans le support dédié) mais contient aussi
des chapitres sur la conception émergente et les pratiques qui prolongent le Clean Code.
Les chapitres 7 à 10 (design OO en pratique) complètent directement cette matinée.
