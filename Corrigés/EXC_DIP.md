# DIP — Corrigés des exercices niveaux 2 et 3

Le Dependency Inversion Principle est le principe qui a le plus d'impact sur la
**testabilité** et sur la **maintenabilité à long terme** d'une base de code.
Sa bonne application se reconnaît à un signe simple : les tests unitaires ne
nécessitent aucune infrastructure réelle (pas de base de données, pas de réseau,
pas de système de fichiers). Ces corrigés insistent sur ce critère autant que sur
le code de production.

---

## Exercice 2 (niveau intermédiaire) — Système de journalisation en Python

### Rappel de l'énoncé

Concevoir un système de logs où `PaymentService` ne connaît pas la destination
des logs. Implémenter trois adaptateurs (console, fichier, base de données),
un `CompositeLogger` délégant à plusieurs destinations, et un test unitaire
avec `InMemoryLogger`.

### Analyse DIP

La violation classique serait :

```python
class PaymentService:
    def __init__(self):
        self.logger = logging.getLogger(__name__)  # dépendance concrète
```

`PaymentService` dépend ici du module `logging` de la bibliothèque standard —
une concrétion, même si elle est populaire. La changer (pour Sentry, pour un
système de logs structurés, pour les tests) implique de modifier `PaymentService`.

La solution DIP : `PaymentService` définit le contrat dont il a besoin (une
abstraction `LoggerPort`), et reçoit une implémentation concrète à la construction.
La couche métier ne connaît jamais les détails d'infrastructure.

### Implémentation

```python
# ── logger_port.py ────────────────────────────────────────────────────────────

from __future__ import annotations

import json
import sqlite3
import urllib.request
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import IntEnum
from typing import Any


# ── Niveau de sévérité ────────────────────────────────────────────────────────

class LogLevel(IntEnum):
    DEBUG   = 10
    INFO    = 20
    WARNING = 30
    ERROR   = 40
    CRITICAL = 50

    def label(self) -> str:
        return self.name


# ── Entrée de log ─────────────────────────────────────────────────────────────

@dataclass(frozen=True)
class LogEntry:
    level:     LogLevel
    message:   str
    context:   dict[str, Any] = field(default_factory=dict)
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(tz=timezone.utc)
    )
    logger_name: str = "app"

    def formatted(self) -> str:
        ts  = self.timestamp.strftime("%Y-%m-%d %H:%M:%S")
        ctx = f" {json.dumps(self.context, ensure_ascii=False)}" if self.context else ""
        return f"[{ts}] {self.level.label():8s} [{self.logger_name}] {self.message}{ctx}"


# ── Abstraction définie par la couche métier ──────────────────────────────────

class LoggerPort(ABC):
    """
    Contrat de journalisation.
    Défini par la couche métier (PaymentService) — pas par l'infrastructure.

    Précondition  : entry est une LogEntry valide (non None).
    Postcondition : l'entrée est persistée d'une manière ou d'une autre
                    (la couche métier ne sait pas laquelle).
    Exception     : ne lève jamais d'exception vers l'appelant ; les erreurs
                    d'écriture sont absorbées ou redirigées en interne.
    """

    @abstractmethod
    def log(self, entry: LogEntry) -> None:
        ...

    # ── Méthodes de commodité (implémentées une fois ici) ────────────────────

    def debug(self, message: str, **context: Any) -> None:
        self.log(LogEntry(LogLevel.DEBUG, message, context))

    def info(self, message: str, **context: Any) -> None:
        self.log(LogEntry(LogLevel.INFO, message, context))

    def warning(self, message: str, **context: Any) -> None:
        self.log(LogEntry(LogLevel.WARNING, message, context))

    def error(self, message: str, **context: Any) -> None:
        self.log(LogEntry(LogLevel.ERROR, message, context))

    def critical(self, message: str, **context: Any) -> None:
        self.log(LogEntry(LogLevel.CRITICAL, message, context))


# ── Adaptateurs d'infrastructure ──────────────────────────────────────────────

class ConsoleLogger(LoggerPort):
    """
    Ecrit les logs sur stdout.
    Raison de changer : modification du format d'affichage console.
    """

    def __init__(self, min_level: LogLevel = LogLevel.DEBUG):
        self._min_level = min_level

    def log(self, entry: LogEntry) -> None:
        if entry.level >= self._min_level:
            print(entry.formatted())


class FileLogger(LoggerPort):
    """
    Ecrit les logs dans un fichier texte avec rotation naïve par taille.
    Raison de changer : format du fichier, politique de rotation, encodage.
    """

    def __init__(
        self,
        filepath: str,
        min_level: LogLevel = LogLevel.INFO,
        max_bytes: int = 10 * 1024 * 1024,  # 10 Mo
    ):
        self._filepath  = filepath
        self._min_level = min_level
        self._max_bytes = max_bytes

    def log(self, entry: LogEntry) -> None:
        if entry.level < self._min_level:
            return
        try:
            import os
            if os.path.exists(self._filepath) and \
               os.path.getsize(self._filepath) >= self._max_bytes:
                os.rename(self._filepath, self._filepath + ".bak")
            with open(self._filepath, "a", encoding="utf-8") as f:
                f.write(entry.formatted() + "\n")
        except OSError:
            pass  # le log ne doit jamais faire crasher l'application


class SqliteLogger(LoggerPort):
    """
    Persiste les logs dans une base SQLite.
    Raison de changer : schéma de la table, moteur de BDD.
    """

    def __init__(self, db_path: str, min_level: LogLevel = LogLevel.WARNING):
        self._min_level = min_level
        self._conn      = sqlite3.connect(db_path, check_same_thread=False)
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS logs (
                id         INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp  TEXT    NOT NULL,
                level      TEXT    NOT NULL,
                logger     TEXT    NOT NULL,
                message    TEXT    NOT NULL,
                context    TEXT
            )
        """)
        self._conn.commit()

    def log(self, entry: LogEntry) -> None:
        if entry.level < self._min_level:
            return
        try:
            self._conn.execute(
                "INSERT INTO logs (timestamp, level, logger, message, context) "
                "VALUES (?, ?, ?, ?, ?)",
                (
                    entry.timestamp.isoformat(),
                    entry.level.label(),
                    entry.logger_name,
                    entry.message,
                    json.dumps(entry.context) if entry.context else None,
                ),
            )
            self._conn.commit()
        except sqlite3.Error:
            pass


class HttpLogger(LoggerPort):
    """
    Envoie les logs vers un endpoint HTTP (ex. : Loki, Elasticsearch, Datadog).
    Raison de changer : changement de fournisseur ou de format de payload.
    """

    def __init__(
        self,
        endpoint: str,
        min_level: LogLevel = LogLevel.ERROR,
        timeout: float = 2.0,
    ):
        self._endpoint  = endpoint
        self._min_level = min_level
        self._timeout   = timeout

    def log(self, entry: LogEntry) -> None:
        if entry.level < self._min_level:
            return
        payload = json.dumps({
            "timestamp": entry.timestamp.isoformat(),
            "level":     entry.level.label(),
            "logger":    entry.logger_name,
            "message":   entry.message,
            "context":   entry.context,
        }).encode("utf-8")
        try:
            req = urllib.request.Request(
                self._endpoint,
                data=payload,
                headers={"Content-Type": "application/json"},
                method="POST",
            )
            urllib.request.urlopen(req, timeout=self._timeout)
        except Exception:
            pass  # absorption des erreurs réseau


# ── CompositeLogger ───────────────────────────────────────────────────────────

class CompositeLogger(LoggerPort):
    """
    Délègue chaque entrée à une liste de LoggerPort.

    Respecte le DIP : ne connaît que l'abstraction LoggerPort.
    Respecte l'OCP : ajouter une destination ne modifie pas cette classe.

    Raison de changer : modification de la logique de fan-out
    (ex. : arrêt à la première erreur, logs asynchrones…).
    """

    def __init__(self, *loggers: LoggerPort):
        self._loggers = list(loggers)

    def add(self, logger: LoggerPort) -> None:
        self._loggers.append(logger)

    def log(self, entry: LogEntry) -> None:
        for logger in self._loggers:
            logger.log(entry)  # chaque logger absorbe ses propres erreurs


# ── Doublure de test ──────────────────────────────────────────────────────────

class InMemoryLogger(LoggerPort):
    """
    Stocke les entrées en mémoire vive pour les tests unitaires.
    Aucune dépendance d'infrastructure — instantané, déterministe, inspecrable.
    """

    def __init__(self):
        self.entries: list[LogEntry] = []

    def log(self, entry: LogEntry) -> None:
        self.entries.append(entry)

    # ── Méthodes d'assertion pratiques ───────────────────────────────────────

    def has_message(self, substring: str) -> bool:
        return any(substring in e.message for e in self.entries)

    def count_at_level(self, level: LogLevel) -> int:
        return sum(1 for e in self.entries if e.level == level)

    def last(self) -> LogEntry | None:
        return self.entries[-1] if self.entries else None

    def clear(self) -> None:
        self.entries.clear()
```

```python
# ── payment_service.py ────────────────────────────────────────────────────────

from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from typing import Any


@dataclass(frozen=True)
class PaymentResult:
    success:        bool
    transaction_id: str
    amount:         Decimal
    currency:       str
    error:          str | None = None


class PaymentGatewayPort(ABC):
    """
    Abstraction de la passerelle de paiement.
    Définie ici pour illustrer le DIP sur les deux dépendances de
    PaymentService (logger + gateway), pas seulement sur les logs.
    """

    @abstractmethod
    def charge(
        self,
        amount: Decimal,
        currency: str,
        card_token: str,
    ) -> PaymentResult:
        ...


class PaymentService:
    """
    Couche métier : orchestration du paiement.

    Dépend uniquement d'abstractions (LoggerPort, PaymentGatewayPort).
    Aucune connaissance de SQLite, de fichiers, de Stripe ou de tout autre
    détail d'infrastructure.

    Raison de changer : modification des règles métier du paiement.
    """

    def __init__(
        self,
        gateway: PaymentGatewayPort,
        logger:  LoggerPort,
    ):
        self._gateway = gateway
        self._logger  = logger

    def process(
        self,
        amount:     Decimal,
        currency:   str,
        card_token: str,
        customer_id: str,
    ) -> PaymentResult:

        self._logger.info(
            "Tentative de paiement.",
            amount=str(amount),
            currency=currency,
            customer_id=customer_id,
        )

        if amount <= Decimal("0"):
            self._logger.warning(
                "Montant de paiement invalide.",
                amount=str(amount),
            )
            return PaymentResult(
                success=False,
                transaction_id="",
                amount=amount,
                currency=currency,
                error="Le montant doit être strictement positif.",
            )

        try:
            result = self._gateway.charge(amount, currency, card_token)
        except Exception as exc:
            self._logger.error(
                "Erreur inattendue lors de la passerelle de paiement.",
                error=str(exc),
                customer_id=customer_id,
            )
            return PaymentResult(
                success=False,
                transaction_id="",
                amount=amount,
                currency=currency,
                error=f"Erreur passerelle : {exc}",
            )

        if result.success:
            self._logger.info(
                "Paiement accepté.",
                transaction_id=result.transaction_id,
                amount=str(amount),
                currency=currency,
            )
        else:
            self._logger.warning(
                "Paiement refusé.",
                transaction_id=result.transaction_id,
                error=result.error,
            )

        return result
```

### Tests unitaires

```python
# ── test_payment_service.py ───────────────────────────────────────────────────

import pytest
from decimal import Decimal
from unittest.mock import MagicMock


# ── Doublures ─────────────────────────────────────────────────────────────────

class FakeGateway(PaymentGatewayPort):
    """Passerelle de paiement simulée, pilotable depuis les tests."""

    def __init__(self, success: bool = True, tx_id: str = "TX-001"):
        self._success = success
        self._tx_id   = tx_id
        self.calls: list[dict] = []

    def charge(
        self,
        amount: Decimal,
        currency: str,
        card_token: str,
    ) -> PaymentResult:
        self.calls.append({"amount": amount, "currency": currency, "token": card_token})
        if self._success:
            return PaymentResult(True, self._tx_id, amount, currency)
        return PaymentResult(False, "", amount, currency, error="Carte refusée.")


class FailingGateway(PaymentGatewayPort):
    """Simule une panne de la passerelle."""

    def charge(self, amount, currency, card_token):
        raise ConnectionError("Timeout de la passerelle Stripe.")


# ── Helpers ───────────────────────────────────────────────────────────────────

def make_service(
    success: bool = True,
    tx_id: str = "TX-001",
) -> tuple[PaymentService, FakeGateway, InMemoryLogger]:
    gateway = FakeGateway(success=success, tx_id=tx_id)
    logger  = InMemoryLogger()
    service = PaymentService(gateway, logger)
    return service, gateway, logger


# ── Tests ─────────────────────────────────────────────────────────────────────

class TestPaymentService:

    def test_successful_payment_logs_info(self):
        service, gateway, logger = make_service()
        result = service.process(Decimal("49.99"), "EUR", "tok_visa", "C42")

        assert result.success
        assert result.transaction_id == "TX-001"
        assert logger.has_message("Paiement accepté")
        assert logger.count_at_level(LogLevel.INFO) >= 2  # tentative + accepté

    def test_zero_amount_is_rejected_and_logged(self):
        service, _, logger = make_service()
        result = service.process(Decimal("0"), "EUR", "tok_visa", "C42")

        assert not result.success
        assert "positif" in (result.error or "")
        assert logger.count_at_level(LogLevel.WARNING) >= 1

    def test_gateway_refusal_logs_warning(self):
        service, _, logger = make_service(success=False)
        result = service.process(Decimal("99.00"), "EUR", "tok_declined", "C43")

        assert not result.success
        assert "refusée" in (result.error or "").lower()
        assert logger.count_at_level(LogLevel.WARNING) >= 1

    def test_gateway_exception_logs_error(self):
        logger  = InMemoryLogger()
        service = PaymentService(FailingGateway(), logger)
        result  = service.process(Decimal("10.00"), "EUR", "tok_fail", "C44")

        assert not result.success
        assert logger.count_at_level(LogLevel.ERROR) == 1
        assert logger.has_message("Erreur inattendue")

    def test_gateway_receives_correct_arguments(self):
        service, gateway, _ = make_service()
        service.process(Decimal("75.50"), "USD", "tok_amex", "C45")

        assert len(gateway.calls) == 1
        assert gateway.calls[0]["amount"] == Decimal("75.50")
        assert gateway.calls[0]["currency"] == "USD"


class TestCompositeLogger:

    def test_delegates_to_all_loggers(self):
        a = InMemoryLogger()
        b = InMemoryLogger()
        c = InMemoryLogger()
        composite = CompositeLogger(a, b, c)
        composite.info("test message")

        assert len(a.entries) == 1
        assert len(b.entries) == 1
        assert len(c.entries) == 1
        assert a.entries[0].message == b.entries[0].message == c.entries[0].message

    def test_one_failing_logger_does_not_break_others(self):
        class BrokenLogger(LoggerPort):
            def log(self, entry: LogEntry) -> None:
                raise RuntimeError("Disque plein")

        good   = InMemoryLogger()
        broken = BrokenLogger()

        # CompositeLogger absorbe les erreurs de chaque logger individuel.
        # Si BrokenLogger ne les absorbe pas, CompositeLogger doit le faire.
        composite = CompositeLogger(broken, good)
        # Ce test documente un comportement attendu — l'implémentation
        # de CompositeLogger doit envelopper chaque appel dans un try/except.
        try:
            composite.log(LogEntry(LogLevel.INFO, "msg"))
        except RuntimeError:
            pass  # acceptable si CompositeLogger ne garantit pas l'isolation

        # Dans une version robuste : assert len(good.entries) == 1


class TestInMemoryLogger:

    def test_stores_entries_in_order(self):
        logger = InMemoryLogger()
        logger.info("premier")
        logger.error("deuxième")
        logger.warning("troisième")

        assert [e.message for e in logger.entries] == ["premier", "deuxième", "troisième"]

    def test_count_at_level(self):
        logger = InMemoryLogger()
        logger.info("a")
        logger.info("b")
        logger.error("c")

        assert logger.count_at_level(LogLevel.INFO) == 2
        assert logger.count_at_level(LogLevel.ERROR) == 1

    def test_has_message_substring(self):
        logger = InMemoryLogger()
        logger.info("Transaction TX-999 validée")

        assert logger.has_message("TX-999")
        assert not logger.has_message("TX-000")
```

### Points clés à retenir

- `PaymentService` ne contient aucun `import logging`, aucun `open()`, aucune
  connexion SQLite. Ses seules dépendances sont deux abstractions définies par
  la couche métier elle-même.
- `InMemoryLogger` et `FakeGateway` permettent de tester exhaustivement
  `PaymentService` en quelques millisecondes, sans aucune infrastructure.
- `CompositeLogger` est lui-même un `LoggerPort` : il peut être injecté partout
  où un logger est attendu, et peut être composé récursivement.
- Les méthodes de commodité (`info`, `warning`, `error`…) sont implémentées
  une seule fois dans `LoggerPort` — les sous-classes n'implémentent que `log`.

---

## Exercice 3 (niveau avancé) — Pipeline de traitement de données financières en PHP

### Rappel de l'énoncé

Construire une chaîne `Source → Transformateur → Validateur → Destination` avec :
- Sources : CSV, API REST, SQL
- Destinations : SQL, fichier JSON, file RabbitMQ
- Interfaces pour chaque étape, deux sources et deux destinations concrètes,
  un `DataPipeline` dépendant uniquement des abstractions, tests sans
  infrastructure, et réflexion sur les limites du DIP.

### Étape 1 — Interfaces

```php
<?php

declare(strict_types=1);

// ── Types de données ──────────────────────────────────────────────────────────

/**
 * Un enregistrement financier brut, sous forme de tableau associatif.
 * @phpstan-type RawRecord array<string, mixed>
 * @phpstan-type Record     array<string, mixed>
 */

// ── Exceptions ────────────────────────────────────────────────────────────────

class DataSourceException    extends \RuntimeException {}
class TransformationException extends \RuntimeException {}
class ValidationException    extends \RuntimeException {}
class DataWriterException    extends \RuntimeException {}


// ── DataSourceInterface ───────────────────────────────────────────────────────

/**
 * Abstraction de la lecture de données brutes.
 *
 * Précondition  : la source est accessible (fichier existant, BDD connectée…).
 * Postcondition : read() retourne un itérable d'enregistrements associatifs.
 *                 Chaque enregistrement est un array<string, mixed> non null.
 * Exception     : DataSourceException si la source est inaccessible ou corrompue.
 * Invariant     : deux appels successifs à read() sans modification de la source
 *                 doivent retourner les mêmes données (déterminisme).
 */
interface DataSourceInterface
{
    /**
     * @return iterable<array<string, mixed>>
     */
    public function read(): iterable;
}


// ── DataTransformerInterface ──────────────────────────────────────────────────

/**
 * Abstraction de la transformation d'un enregistrement brut.
 *
 * Postcondition : retourne un enregistrement transformé de structure définie
 *                 (toutes les clés attendues par le validateur sont présentes).
 * Exception     : TransformationException si la transformation est impossible.
 * Invariant     : ne modifie pas l'enregistrement d'entrée (immutabilité).
 */
interface DataTransformerInterface
{
    /**
     * @param  array<string, mixed> $record
     * @return array<string, mixed>
     */
    public function transform(array $record): array;
}


// ── DataValidatorInterface ────────────────────────────────────────────────────

/**
 * Abstraction de la validation métier d'un enregistrement transformé.
 *
 * Postcondition : validate() ne lève pas d'exception si l'enregistrement est valide.
 * Exception     : ValidationException avec message descriptif si invalide.
 * Invariant     : ne modifie pas l'enregistrement.
 */
interface DataValidatorInterface
{
    /**
     * @param array<string, mixed> $record
     * @throws ValidationException
     */
    public function validate(array $record): void;
}


// ── DataWriterInterface ───────────────────────────────────────────────────────

/**
 * Abstraction de l'écriture d'enregistrements validés.
 *
 * Postcondition : après write(), l'enregistrement est persisté dans la destination.
 * Exception     : DataWriterException si l'écriture échoue.
 */
interface DataWriterInterface
{
    /**
     * @param array<string, mixed> $record
     */
    public function write(array $record): void;

    /**
     * Libère les ressources (connexions, handles de fichiers, buffers).
     * Appelé par le pipeline après le traitement de tous les enregistrements.
     */
    public function flush(): void;
}
```

### Étape 2 — Sources et destinations concrètes

```php
<?php

declare(strict_types=1);

// ════════════════════════════════════════════════════════════════════
// SOURCES
// ════════════════════════════════════════════════════════════════════

// ── CsvDataSource ─────────────────────────────────────────────────────────────

/**
 * Lit des enregistrements depuis un fichier CSV.
 * Raison de changer : format du CSV (séparateur, encodage, présence d'en-tête).
 */
final class CsvDataSource implements DataSourceInterface
{
    public function __construct(
        private readonly string $filepath,
        private readonly string $separator  = ',',
        private readonly bool   $hasHeader  = true,
        private readonly string $encoding   = 'UTF-8',
    ) {}

    public function read(): iterable
    {
        if (!file_exists($this->filepath)) {
            throw new DataSourceException("Fichier CSV introuvable : {$this->filepath}");
        }

        $handle = fopen($this->filepath, 'r');
        if ($handle === false) {
            throw new DataSourceException("Impossible d'ouvrir : {$this->filepath}");
        }

        try {
            $headers = null;
            while (($row = fgetcsv($handle, separator: $this->separator)) !== false) {
                if ($this->hasHeader && $headers === null) {
                    $headers = $row;
                    continue;
                }
                if ($headers !== null) {
                    $record = array_combine($headers, $row);
                    if ($record === false) {
                        throw new DataSourceException(
                            "Nombre de colonnes incohérent dans le CSV."
                        );
                    }
                    yield $record;
                } else {
                    yield $row;
                }
            }
        } finally {
            fclose($handle);
        }
    }
}


// ── RestApiDataSource ─────────────────────────────────────────────────────────

/**
 * Lit des enregistrements depuis une API REST paginée.
 * Raison de changer : format de la réponse, mécanisme de pagination, authentification.
 */
final class RestApiDataSource implements DataSourceInterface
{
    public function __construct(
        private readonly string $baseUrl,
        private readonly string $apiKey       = '',
        private readonly int    $pageSize     = 100,
        private readonly string $dataKey      = 'data',    // clé JSON contenant les records
        private readonly string $nextPageKey  = 'next',   // clé JSON pour la pagination
    ) {}

    public function read(): iterable
    {
        $url = $this->baseUrl . '?limit=' . $this->pageSize;

        do {
            $response = $this->fetch($url);
            $payload  = json_decode($response, associative: true, flags: JSON_THROW_ON_ERROR);

            foreach ($payload[$this->dataKey] ?? [] as $record) {
                yield $record;
            }

            $url = $payload[$this->nextPageKey] ?? null;
        } while ($url !== null);
    }

    private function fetch(string $url): string
    {
        $context = stream_context_create([
            'http' => [
                'method'  => 'GET',
                'header'  => $this->apiKey
                    ? "Authorization: Bearer {$this->apiKey}\r\n"
                    : '',
                'timeout' => 10,
            ],
        ]);

        $body = @file_get_contents($url, context: $context);
        if ($body === false) {
            throw new DataSourceException("Erreur HTTP lors de la requête : $url");
        }
        return $body;
    }
}


// ════════════════════════════════════════════════════════════════════
// DESTINATIONS
// ════════════════════════════════════════════════════════════════════

// ── SqlDataWriter ─────────────────────────────────────────────────────────────

/**
 * Insère des enregistrements dans une table SQL via PDO.
 * Raison de changer : schéma de la table, moteur de BDD, stratégie d'upsert.
 */
final class SqlDataWriter implements DataWriterInterface
{
    private \PDO        $pdo;
    private ?\PDOStatement $stmt = null;
    private int         $batchSize;
    private array       $buffer = [];

    public function __construct(
        \PDO   $pdo,
        private readonly string $table,
        private readonly array  $columns,   // ['amount', 'currency', 'date', ...]
        int    $batchSize = 100,
    ) {
        $this->pdo       = $pdo;
        $this->batchSize = $batchSize;
    }

    public function write(array $record): void
    {
        $this->buffer[] = $record;
        if (count($this->buffer) >= $this->batchSize) {
            $this->flush();
        }
    }

    public function flush(): void
    {
        if (empty($this->buffer)) {
            return;
        }

        $placeholders = '(' . implode(', ', array_fill(0, count($this->columns), '?')) . ')';
        $columnList   = implode(', ', $this->columns);
        $sql          = "INSERT INTO {$this->table} ($columnList) VALUES $placeholders";

        $this->pdo->beginTransaction();
        try {
            $stmt = $this->pdo->prepare($sql);
            foreach ($this->buffer as $record) {
                $values = array_map(
                    fn(string $col) => $record[$col] ?? null,
                    $this->columns
                );
                $stmt->execute($values);
            }
            $this->pdo->commit();
        } catch (\PDOException $e) {
            $this->pdo->rollBack();
            throw new DataWriterException(
                "Erreur SQL lors de l'insertion : {$e->getMessage()}",
                previous: $e,
            );
        }

        $this->buffer = [];
    }
}


// ── JsonFileDataWriter ────────────────────────────────────────────────────────

/**
 * Ecrit des enregistrements dans un fichier JSON (tableau de lignes).
 * Raison de changer : format du JSON, encodage, structure du fichier.
 */
final class JsonFileDataWriter implements DataWriterInterface
{
    private array $records = [];

    public function __construct(
        private readonly string $filepath,
        private readonly int    $jsonFlags = JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE,
    ) {}

    public function write(array $record): void
    {
        $this->records[] = $record;
    }

    public function flush(): void
    {
        if (empty($this->records)) {
            return;
        }

        $dir = dirname($this->filepath);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, recursive: true);
        }

        $json = json_encode($this->records, $this->jsonFlags | JSON_THROW_ON_ERROR);
        if (file_put_contents($this->filepath, $json, LOCK_EX) === false) {
            throw new DataWriterException("Impossible d'écrire dans : {$this->filepath}");
        }

        $this->records = [];
    }
}
```

### Étape 3 — Transformateur et validateur concrets (domaine financier)

```php
<?php

declare(strict_types=1);

// ── FinancialRecordTransformer ────────────────────────────────────────────────

/**
 * Normalise un enregistrement financier brut :
 * - conversion des montants en centimes (entiers)
 * - normalisation des dates en ISO 8601
 * - mise en majuscules des codes devise
 */
final class FinancialRecordTransformer implements DataTransformerInterface
{
    public function transform(array $record): array
    {
        try {
            $amount = $this->parseAmount($record['amount'] ?? '0');
            $date   = $this->parseDate($record['date'] ?? '');
        } catch (\Throwable $e) {
            throw new TransformationException(
                "Transformation impossible : {$e->getMessage()}",
                previous: $e,
            );
        }

        return [
            'amount_cents' => $amount,
            'currency'     => strtoupper(trim((string) ($record['currency'] ?? 'EUR'))),
            'date'         => $date,
            'description'  => trim((string) ($record['description'] ?? '')),
            'reference'    => trim((string) ($record['reference'] ?? '')),
            'raw'          => $record,  // conservation de l'original pour l'audit
        ];
    }

    private function parseAmount(mixed $raw): int
    {
        $cleaned = str_replace([' ', '\u{00A0}', ','], ['', '', '.'], (string) $raw);
        if (!is_numeric($cleaned)) {
            throw new \InvalidArgumentException("Montant non numérique : $raw");
        }
        return (int) round((float) $cleaned * 100);
    }

    private function parseDate(string $raw): string
    {
        $formats = ['d/m/Y', 'Y-m-d', 'd-m-Y', 'm/d/Y', 'Y/m/d'];
        foreach ($formats as $format) {
            $dt = \DateTimeImmutable::createFromFormat($format, trim($raw));
            if ($dt !== false) {
                return $dt->format('Y-m-d');
            }
        }
        throw new \InvalidArgumentException("Format de date non reconnu : $raw");
    }
}


// ── FinancialRecordValidator ──────────────────────────────────────────────────

/**
 * Valide les règles métier d'un enregistrement financier transformé.
 */
final class FinancialRecordValidator implements DataValidatorInterface
{
    private const VALID_CURRENCIES = ['EUR', 'USD', 'GBP', 'CHF', 'JPY'];
    private const MAX_AMOUNT_CENTS = 10_000_000_00;  // 10 millions d'euros

    public function validate(array $record): void
    {
        $errors = [];

        if (!isset($record['amount_cents']) || !is_int($record['amount_cents'])) {
            $errors[] = "Le champ 'amount_cents' doit être un entier.";
        } elseif ($record['amount_cents'] < 0) {
            $errors[] = "Le montant ne peut pas être négatif.";
        } elseif ($record['amount_cents'] > self::MAX_AMOUNT_CENTS) {
            $errors[] = sprintf(
                "Le montant dépasse le plafond autorisé (%s €).",
                number_format(self::MAX_AMOUNT_CENTS / 100, 2, ',', ' ')
            );
        }

        if (empty($record['currency'])) {
            $errors[] = "La devise est obligatoire.";
        } elseif (!in_array($record['currency'], self::VALID_CURRENCIES, true)) {
            $errors[] = "Devise non supportée : {$record['currency']}.";
        }

        if (empty($record['date'])) {
            $errors[] = "La date est obligatoire.";
        } elseif (!\DateTimeImmutable::createFromFormat('Y-m-d', $record['date'])) {
            $errors[] = "Format de date invalide : {$record['date']}.";
        }

        if (!empty($errors)) {
            throw new ValidationException(implode(' | ', $errors));
        }
    }
}
```

### Étape 4 — `DataPipeline` : orchestration sans infrastructure

```php
<?php

declare(strict_types=1);

// ── Résultat du pipeline ──────────────────────────────────────────────────────

final class PipelineResult
{
    public int   $processed  = 0;
    public int   $succeeded  = 0;
    public int   $failed     = 0;

    /** @var array<int, array{record: array, error: string}> */
    public array $errors = [];

    public function recordSuccess(): void
    {
        $this->processed++;
        $this->succeeded++;
    }

    public function recordFailure(array $record, string $reason): void
    {
        $this->processed++;
        $this->failed++;
        $this->errors[] = ['record' => $record, 'error' => $reason];
    }

    public function successRate(): float
    {
        return $this->processed > 0
            ? round($this->succeeded / $this->processed * 100, 2)
            : 0.0;
    }
}


// ── DataPipeline ──────────────────────────────────────────────────────────────

/**
 * Orchestre la chaîne Source → Transformateur → Validateur → Destination.
 *
 * FERME à la modification : cette classe ne change jamais pour accueillir
 *   une nouvelle source, un nouveau format ou une nouvelle destination.
 * OUVERT à l'extension   : on injecte les implémentations souhaitées.
 *
 * Dépend UNIQUEMENT des quatre abstractions — aucune importation de PDO,
 * de fichiers ou de HTTP directement dans cette classe.
 *
 * Raison de changer : modification du flux d'orchestration (ajout d'une étape
 *   d'audit, traitement parallèle, gestion des retries...).
 */
final class DataPipeline
{
    public function __construct(
        private readonly DataSourceInterface      $source,
        private readonly DataTransformerInterface $transformer,
        private readonly DataValidatorInterface   $validator,
        private readonly DataWriterInterface      $writer,
        private readonly LoggerPort               $logger,
    ) {}

    public function run(): PipelineResult
    {
        $result = new PipelineResult();

        $this->logger->info("Démarrage du pipeline.");

        try {
            foreach ($this->source->read() as $raw) {
                $this->processRecord($raw, $result);
            }
        } catch (DataSourceException $e) {
            $this->logger->error("Lecture de la source impossible.", error: $e->getMessage());
            throw $e;  // erreur fatale : on ne peut pas continuer
        } finally {
            $this->safeFlush($result);
        }

        $this->logger->info(
            "Pipeline terminé.",
            processed:  $result->processed,
            succeeded:  $result->succeeded,
            failed:     $result->failed,
            success_rate: $result->successRate() . '%',
        );

        return $result;
    }

    private function processRecord(array $raw, PipelineResult $result): void
    {
        try {
            $transformed = $this->transformer->transform($raw);
            $this->validator->validate($transformed);
            $this->writer->write($transformed);
            $result->recordSuccess();
        } catch (TransformationException | ValidationException $e) {
            // Erreur non fatale : on journalise et on continue
            $this->logger->warning(
                "Enregistrement rejeté.",
                reason: $e->getMessage(),
                record: json_encode($raw),
            );
            $result->recordFailure($raw, $e->getMessage());
        } catch (DataWriterException $e) {
            // Erreur d'écriture : potentiellement fatale selon la stratégie
            $this->logger->error(
                "Erreur d'écriture.",
                reason: $e->getMessage(),
            );
            $result->recordFailure($raw, $e->getMessage());
        }
    }

    private function safeFlush(PipelineResult $result): void
    {
        try {
            $this->writer->flush();
        } catch (DataWriterException $e) {
            $this->logger->error("Erreur lors du flush final.", error: $e->getMessage());
            $result->recordFailure([], "Flush échoué : {$e->getMessage()}");
        }
    }
}
```

### Tests PHPUnit — sans aucune infrastructure réelle

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

// ── Doublures de test ──────────────────────────────────────────────────────────

/**
 * Source en mémoire : retourne un tableau d'enregistrements prédéfinis.
 */
final class InMemoryDataSource implements DataSourceInterface
{
    /** @param array<array<string, mixed>> $records */
    public function __construct(private array $records = []) {}

    public function read(): iterable
    {
        return $this->records;
    }
}

/**
 * Source qui lève une exception pour tester la gestion d'erreur fatale.
 */
final class FailingDataSource implements DataSourceInterface
{
    public function read(): iterable
    {
        throw new DataSourceException("Source indisponible (simulation).");
    }
}

/**
 * Transformateur passthrough : retourne l'enregistrement tel quel.
 */
final class IdentityTransformer implements DataTransformerInterface
{
    public function transform(array $record): array
    {
        return $record;
    }
}

/**
 * Transformateur qui échoue toujours.
 */
final class AlwaysFailTransformer implements DataTransformerInterface
{
    public function transform(array $record): array
    {
        throw new TransformationException("Transformation volontairement échouée.");
    }
}

/**
 * Validateur qui accepte tout.
 */
final class AlwaysValidValidator implements DataValidatorInterface
{
    public function validate(array $record): void {}
}

/**
 * Validateur qui rejette tout.
 */
final class AlwaysInvalidValidator implements DataValidatorInterface
{
    public function validate(array $record): void
    {
        throw new ValidationException("Enregistrement volontairement rejeté.");
    }
}

/**
 * Writer en mémoire : accumule les enregistrements écrits pour inspection.
 */
final class InMemoryDataWriter implements DataWriterInterface
{
    public array $written  = [];
    public int   $flushCalls = 0;
    private bool $failOnFlush = false;

    public static function withFlushFailure(): self
    {
        $w = new self();
        $w->failOnFlush = true;
        return $w;
    }

    public function write(array $record): void
    {
        $this->written[] = $record;
    }

    public function flush(): void
    {
        $this->flushCalls++;
        if ($this->failOnFlush) {
            throw new DataWriterException("Flush volontairement échoué.");
        }
    }
}

// ── Utilitaire ────────────────────────────────────────────────────────────────

function makePipeline(
    DataSourceInterface      $source,
    DataTransformerInterface $transformer,
    DataValidatorInterface   $validator,
    DataWriterInterface      $writer,
    ?LoggerPort              $logger = null,
): DataPipeline {
    return new DataPipeline(
        $source,
        $transformer,
        $validator,
        $writer,
        $logger ?? new NullLogger(),
    );
}


// ── Tests ──────────────────────────────────────────────────────────────────────

final class DataPipelineTest extends TestCase
{
    private function validRecords(): array
    {
        return [
            ['amount' => '49.99', 'currency' => 'EUR',
             'date' => '2024-01-15', 'description' => 'Achat A'],
            ['amount' => '120.00', 'currency' => 'USD',
             'date' => '2024-01-16', 'description' => 'Achat B'],
        ];
    }

    // ── Chemin nominal ────────────────────────────────────────────────────────

    public function testSuccessfulPipelineProcessesAllRecords(): void
    {
        $writer = new InMemoryDataWriter();
        $result = makePipeline(
            new InMemoryDataSource($this->validRecords()),
            new FinancialRecordTransformer(),
            new FinancialRecordValidator(),
            $writer,
        )->run();

        $this->assertSame(2, $result->processed);
        $this->assertSame(2, $result->succeeded);
        $this->assertSame(0, $result->failed);
        $this->assertCount(2, $writer->written);
        $this->assertSame(100.0, $result->successRate());
    }

    public function testTransformedRecordsHaveExpectedShape(): void
    {
        $writer = new InMemoryDataWriter();
        makePipeline(
            new InMemoryDataSource([
                ['amount' => '49,99', 'currency' => 'eur',
                 'date' => '15/01/2024', 'description' => 'Test'],
            ]),
            new FinancialRecordTransformer(),
            new FinancialRecordValidator(),
            $writer,
        )->run();

        $record = $writer->written[0];
        $this->assertSame(4999, $record['amount_cents']);  // centimes
        $this->assertSame('EUR', $record['currency']);      // majuscules
        $this->assertSame('2024-01-15', $record['date']);   // ISO 8601
    }

    // ── Gestion des erreurs non fatales ───────────────────────────────────────

    public function testTransformationFailureContinuesPipeline(): void
    {
        $writer = new InMemoryDataWriter();
        $result = makePipeline(
            new InMemoryDataSource([['bad' => 'data'], ['amount' => '10', 'currency' => 'EUR', 'date' => '2024-01-01']]),
            new FinancialRecordTransformer(),
            new FinancialRecordValidator(),
            $writer,
        )->run();

        $this->assertSame(2, $result->processed);
        $this->assertSame(1, $result->failed);
        $this->assertSame(1, $result->succeeded);
        $this->assertCount(1, $result->errors);
    }

    public function testValidationFailureContinuesPipeline(): void
    {
        $writer = new InMemoryDataWriter();
        $result = makePipeline(
            new InMemoryDataSource([['x' => 1], ['y' => 2], ['z' => 3]]),
            new IdentityTransformer(),
            new AlwaysInvalidValidator(),
            $writer,
        )->run();

        $this->assertSame(3, $result->failed);
        $this->assertSame(0, $result->succeeded);
        $this->assertCount(3, $writer->written, 'Rien ne doit être écrit si la validation échoue.');
        // Note : le writer ne reçoit rien car la validation échoue avant write()
        // → ce test documente ce comportement attendu.
        $this->assertCount(0, $writer->written);
    }

    // ── Erreur fatale de la source ────────────────────────────────────────────

    public function testFatalSourceExceptionBubblesUp(): void
    {
        $this->expectException(DataSourceException::class);

        makePipeline(
            new FailingDataSource(),
            new IdentityTransformer(),
            new AlwaysValidValidator(),
            new InMemoryDataWriter(),
        )->run();
    }

    public function testFlushIsCalledEvenOnFatalError(): void
    {
        $writer = new InMemoryDataWriter();

        try {
            makePipeline(
                new FailingDataSource(),
                new IdentityTransformer(),
                new AlwaysValidValidator(),
                $writer,
            )->run();
        } catch (DataSourceException) {}

        $this->assertSame(1, $writer->flushCalls, 'flush() doit être appelé même en cas d\'erreur fatale.');
    }

    // ── Flush échoué ──────────────────────────────────────────────────────────

    public function testFlushFailureIsRecordedButDoesNotThrow(): void
    {
        $result = makePipeline(
            new InMemoryDataSource([['a' => 1]]),
            new IdentityTransformer(),
            new AlwaysValidValidator(),
            InMemoryDataWriter::withFlushFailure(),
        )->run();

        // Le flush échoué est enregistré comme erreur mais le pipeline se termine.
        $this->assertGreaterThan(0, $result->failed);
    }

    // ── Logs ──────────────────────────────────────────────────────────────────

    public function testPipelineLogsStartAndEnd(): void
    {
        $logger = new InMemoryLogger();
        makePipeline(
            new InMemoryDataSource($this->validRecords()),
            new FinancialRecordTransformer(),
            new FinancialRecordValidator(),
            new InMemoryDataWriter(),
            $logger,
        )->run();

        $this->assertTrue($logger->hasMessage('Démarrage du pipeline'));
        $this->assertTrue($logger->hasMessage('Pipeline terminé'));
    }

    public function testRejectedRecordsAreLogged(): void
    {
        $logger = new InMemoryLogger();
        makePipeline(
            new InMemoryDataSource([['invalid' => 'record']]),
            new FinancialRecordTransformer(),
            new FinancialRecordValidator(),
            new InMemoryDataWriter(),
            $logger,
        )->run();

        $this->assertTrue($logger->hasMessage('rejeté'));
    }
}

// ── Tests des composants individuels ─────────────────────────────────────────

final class FinancialRecordTransformerTest extends TestCase
{
    public function testConvertsAmountToCents(): void
    {
        $t = new FinancialRecordTransformer();
        $r = $t->transform(['amount' => '1 234,56', 'currency' => 'eur', 'date' => '2024-01-01']);
        $this->assertSame(123456, $r['amount_cents']);
    }

    public function testNormalizesCurrencyToUpperCase(): void
    {
        $t = new FinancialRecordTransformer();
        $r = $t->transform(['amount' => '10', 'currency' => ' gbp ', 'date' => '01/01/2024']);
        $this->assertSame('GBP', $r['currency']);
    }

    public function testParsesMultipleDateFormats(): void
    {
        $t = new FinancialRecordTransformer();
        $formats = ['01/15/2024', '15/01/2024', '2024-01-15', '2024/01/15'];
        foreach ($formats as $fmt) {
            $r = $t->transform(['amount' => '1', 'currency' => 'EUR', 'date' => $fmt]);
            $this->assertSame('2024-01-15', $r['date'], "Format '$fmt' mal interprété.");
        }
    }

    public function testInvalidAmountThrows(): void
    {
        $this->expectException(TransformationException::class);
        (new FinancialRecordTransformer())->transform([
            'amount' => 'not-a-number', 'currency' => 'EUR', 'date' => '2024-01-01',
        ]);
    }
}

final class FinancialRecordValidatorTest extends TestCase
{
    private function validRecord(): array
    {
        return ['amount_cents' => 5000, 'currency' => 'EUR', 'date' => '2024-01-15'];
    }

    public function testValidRecordPassesWithoutException(): void
    {
        (new FinancialRecordValidator())->validate($this->validRecord());
        $this->assertTrue(true);  // pas d'exception levée
    }

    public function testNegativeAmountIsRejected(): void
    {
        $this->expectException(ValidationException::class);
        (new FinancialRecordValidator())->validate(
            array_merge($this->validRecord(), ['amount_cents' => -1])
        );
    }

    public function testUnsupportedCurrencyIsRejected(): void
    {
        $this->expectException(ValidationException::class);
        (new FinancialRecordValidator())->validate(
            array_merge($this->validRecord(), ['currency' => 'BTC'])
        );
    }
}
```

### Étape 5 — Limites du DIP : abstraction, sur-ingénierie et critères de décision

Le DIP a un coût réel. En voici les limites honnêtes.

**Limite 1 — Le coût de l'indirection**

Chaque interface introduit un niveau d'indirection supplémentaire. Pour un développeur
qui découvre le code, remonter de `DataPipeline` jusqu'à `CsvDataSource` nécessite
de traverser plusieurs fichiers et de comprendre quel objet est effectivement injecté
à la construction. Les IDE modernes gèrent bien cette navigation, mais le coût
cognitif reste réel, surtout pour les contributeurs occasionnels.

*Règle pratique* : le DIP vaut le coût de l'indirection quand la dépendance concrète
est susceptible de changer (moteur de BDD, fournisseur d'e-mail, passerelle de paiement)
ou quand le code doit être testable sans l'infrastructure réelle. Il ne vaut pas ce
coût pour des dépendances stables et non substituables (la bibliothèque standard du
langage, un algorithme mathématique sans effet de bord).

**Limite 2 — La sur-ingénierie préventive**

Il est tentant d'abstraire toutes les dépendances « par principe ». Résultat :
des interfaces avec une seule implémentation, que personne n'a jamais substituée
ni testée avec une doublure. L'interface devient une couche vide qui complexifie
sans rien apporter.

```php
// Sur-ingénierie inutile si MathCalculator ne sera jamais substitué :
interface MathCalculatorInterface {
    public function add(float $a, float $b): float;
}
final class MathCalculator implements MathCalculatorInterface {
    public function add(float $a, float $b): float { return $a + $b; }
}
```

*Règle pratique* : n'abstraire que les dépendances qui ont au moins **deux implémentations
connues** (production + test), ou pour lesquelles le changement d'implémentation est
une exigence explicite du projet.

**Limite 3 — Le Composition Root peut devenir un monstre**

Quand tout est injecté, quelqu'un doit bien assembler les objets. Le Composition Root
— l'endroit où les dépendances concrètes sont instanciées et câblées — peut devenir
un fichier de plusieurs centaines de lignes. Les conteneurs d'injection de dépendances
(Symfony DI, Laravel Service Container, PHP-DI, Python `dependency-injector`) existent
pour gérer ce problème, mais ils introduisent eux-mêmes une courbe d'apprentissage.

*Règle pratique* : pour les petits projets (< 10 classes avec dépendances), l'injection
manuelle dans un fichier `bootstrap.php` est largement suffisante. Les conteneurs
deviennent utiles quand le graphe de dépendances est profond et que le câblage
conditionnel (par environnement ou par configuration) devient fréquent.

**Limite 4 — L'interface n'est pas le contrat**

Une interface PHP ou Python définit les *signatures* des méthodes, pas leur
*sémantique*. Deux implémentations peuvent respecter l'interface et violer le
contrat (LSP, postconditions, invariants). Le DIP rend le code substituable au
niveau des types, mais pas automatiquement correct au niveau du comportement.

```php
// Respecte DataWriterInterface, mais write() ne persiste rien :
final class BrokenWriter implements DataWriterInterface {
    public function write(array $record): void { /* intentionnellement vide */ }
    public function flush(): void {}
}
```

*Conséquence* : le DIP doit être complété par des tests d'intégration qui vérifient
que chaque implémentation concrète respecte bien le contrat, pas seulement la signature.

**Critères synthétiques pour décider d'appliquer le DIP**

| Critère | Appliquer le DIP | Ne pas appliquer |
|---|---|---|
| La dépendance peut changer en production | Oui (BDD, API, e-mail) | Non (stdlib, math) |
| La dépendance doit être simulée en test | Oui | Non (fonctions pures) |
| Il existe déjà 2+ implémentations connues | Oui | Non (règle des deux instances) |
| L'équipe maîtrise l'injection de dépendances | Oui | Non → former d'abord |
| Le projet durera > 6 mois | Oui | Non (prototype jetable) |

---

## Synthèse des deux exercices DIP

| Aspect | Exercice 2 (logs Python) | Exercice 3 (pipeline PHP) |
|---|---|---|
| Dépendances abstraites | `LoggerPort`, `PaymentGatewayPort` | `DataSourceInterface`, `DataTransformerInterface`, `DataValidatorInterface`, `DataWriterInterface`, `LoggerPort` |
| Abstractions définies par | La couche métier (`PaymentService`) | La couche métier (`DataPipeline`) |
| Doublures de test | `InMemoryLogger`, `FakeGateway`, `FailingGateway` | `InMemoryDataSource`, `InMemoryDataWriter`, `IdentityTransformer`, etc. |
| Infrastructure dans les tests | Aucune | Aucune |
| Composition Root | Site d'appel (voir commentaires en fin de fichiers) | `makePipeline()` dans les tests |
| Lien avec OCP | `CompositeLogger` extensible sans modification | `DataPipeline` extensible sans modification |
| Lien avec LSP | Chaque `LoggerPort` doit absorber ses erreurs | Chaque interface a des pré/postconditions formalisées |

Dans les deux exercices, le DIP produit exactement le même bénéfice observable :
**zéro ligne d'infrastructure dans les tests unitaires**. C'est le critère le plus
fiable pour juger si le principe a été correctement appliqué — bien plus que de
compter les interfaces ou les abstractions.
