# LSP — Corrigés des exercices niveaux 2 et 3

Le Liskov Substitution Principle est probablement le plus subtil des cinq principes
SOLID : sa violation ne produit pas toujours une erreur immédiate, mais un
comportement inattendu qui se manifeste à l'exécution, souvent loin du site de
l'appel. Ces corrigés insistent donc sur la **formalisation des contrats** avant
toute ligne de code.

# Exercice 2 (niveau intermédiaire) — Hiérarchie de stockage de fichiers en Python

## Rappel de l'énoncé

Partir de `FileStorage` (quatre méthodes : `read`, `write`, `delete`, `list_files`)
et introduire `S3Storage`, `ReadOnlyLocalStorage` et `EncryptedStorage` sans violer
le LSP.

### Étape 1 — Formalisation du contrat de `FileStorage`

Avant de dessiner la hiérarchie, il faut préciser ce que chaque méthode *garantit*
(postconditions) et ce qu'elle *exige* (préconditions). C'est ce contrat que tout
sous-type devra respecter.

```
read(path)
  Préconditions  : path est une chaîne non vide ; le fichier existe.
  Postconditions : retourne les octets du fichier ; ne modifie aucun état.
  Exceptions     : FileNotFoundError si le fichier est absent.

write(path, data)
  Préconditions  : path est une chaîne non vide ; data est un bytes.
  Postconditions : les octets sont persistés ; read(path) == data après l'appel.
  Exceptions     : PermissionError si le support est en lecture seule (contrat
                   établi explicitement pour éviter les NotImplementedError muettes).

delete(path)
  Préconditions  : path est une chaîne non vide ; le fichier existe.
  Postconditions : le fichier n'existe plus ; list_files() ne le contient plus.
  Exceptions     : FileNotFoundError si le fichier est absent.

list_files()
  Préconditions  : aucune.
  Postconditions : retourne la liste (éventuellement vide) de tous les chemins
                   accessibles en lecture.
  Exceptions     : aucune.
```

### Étape 2 — Analyse des violations potentielles

| Classe | Problème si elle hérite directement de `FileStorage` |
|---|---|
| `S3Storage` | Aucun problème a priori : S3 supporte read, write, delete, list. La précondition sur `path` change de forme (clé S3 ≠ chemin Unix), mais le *contrat observable* peut être préservé. |
| `ReadOnlyLocalStorage` | Violation directe : `write` et `delete` ne peuvent pas satisfaire leur postcondition. Lever `NotImplementedError` affaiblit les postconditions — le code client qui suppose que `write` fonctionne sera trompé. |
| `EncryptedStorage` | Risque de violation sur `read` si le déchiffrement échoue silencieusement ou retourne des données corrompues. La postcondition `read(path) == data écrit` devient `read(path) == decrypt(encrypt(data))` — valide *si et seulement si* le chiffrement est déterministe et symétrique. |

`ReadOnlyLocalStorage` est le cas problématique : elle *ne peut pas* honorer le
contrat de `write` et `delete`. La solution n'est pas de lever une exception muette,
mais de **revoir la hiérarchie** pour que cette classe n'hérite pas d'une interface
qui lui promet ces capacités.

### Étape 3 — Hiérarchie d'interfaces respectant le LSP

```
ReadableStorage          WritableStorage         ListableStorage
  read(path) -> bytes      write(path, data)        list_files() -> list[str]
                           delete(path)

            ↑                    ↑                        ↑
            └────────────────────┴────────────────────────┘
                                 │
                        ReadWriteStorage
                  (combine les trois contrats)

Implémentations :
  LocalStorage         → ReadWriteStorage  (lecture + écriture + liste)
  S3Storage            → ReadWriteStorage
  ReadOnlyLocalStorage → ReadableStorage + ListableStorage  (pas WritableStorage)
  EncryptedStorage     → ReadWriteStorage  (décore un ReadWriteStorage existant)
```

### Implémentation complète

```python
# ── storage_interfaces.py ─────────────────────────────────────────────────────

from __future__ import annotations

import os
import shutil
from abc import ABC, abstractmethod


class ReadableStorage(ABC):
    """
    Contrat : lire des octets à partir d'un chemin.

    Préconditions  : path non vide, fichier existant.
    Postconditions : retourne exactement les octets du fichier.
    Exceptions     : FileNotFoundError si absent.
    """

    @abstractmethod
    def read(self, path: str) -> bytes:
        ...


class WritableStorage(ABC):
    """
    Contrat : persister et supprimer des octets.

    write — postcondition : read(path) == data immédiatement après l'appel.
    delete — postcondition : le chemin disparaît de list_files().
    Exceptions : PermissionError si le support refuse l'opération.
    """

    @abstractmethod
    def write(self, path: str, data: bytes) -> None:
        ...

    @abstractmethod
    def delete(self, path: str) -> None:
        ...


class ListableStorage(ABC):
    """
    Contrat : énumérer les chemins disponibles.

    Postconditions : résultat non None, peut être vide, jamais d'exception.
    """

    @abstractmethod
    def list_files(self) -> list[str]:
        ...


class ReadWriteStorage(ReadableStorage, WritableStorage, ListableStorage, ABC):
    """
    Interface composite pour les supports qui supportent toutes les opérations.
    Hériter de cette interface engage à satisfaire les trois contrats.
    """
    ...


# ── local_storage.py ──────────────────────────────────────────────────────────

class LocalStorage(ReadWriteStorage):
    """
    Stockage local sur disque. Respecte intégralement les trois contrats.
    Raison de changer : évolution du système de fichiers ou des permissions.
    """

    def __init__(self, base_dir: str):
        self._base = base_dir
        os.makedirs(base_dir, exist_ok=True)

    def _full_path(self, path: str) -> str:
        # Sécurité minimale : on interdit les traversées de répertoire.
        normalized = os.path.normpath(path).lstrip("/")
        return os.path.join(self._base, normalized)

    def read(self, path: str) -> bytes:
        full = self._full_path(path)
        if not os.path.exists(full):
            raise FileNotFoundError(f"Fichier introuvable : {path}")
        with open(full, "rb") as f:
            return f.read()

    def write(self, path: str, data: bytes) -> None:
        full = self._full_path(path)
        os.makedirs(os.path.dirname(full), exist_ok=True)
        with open(full, "wb") as f:
            f.write(data)

    def delete(self, path: str) -> None:
        full = self._full_path(path)
        if not os.path.exists(full):
            raise FileNotFoundError(f"Fichier introuvable : {path}")
        os.remove(full)

    def list_files(self) -> list[str]:
        result = []
        for root, _, files in os.walk(self._base):
            for name in files:
                abs_path = os.path.join(root, name)
                result.append(os.path.relpath(abs_path, self._base))
        return result


# ── read_only_local_storage.py ────────────────────────────────────────────────

class ReadOnlyLocalStorage(ReadableStorage, ListableStorage):
    """
    Stockage local en lecture seule.

    N'implémente PAS WritableStorage : aucun risque de violer le LSP
    sur write() ou delete(), car ces méthodes ne font pas partie de
    son contrat.

    Un client qui a besoin d'écrire demandera ReadWriteStorage ou
    WritableStorage — il ne recevra jamais un ReadOnlyLocalStorage.
    """

    def __init__(self, base_dir: str):
        self._base = base_dir

    def _full_path(self, path: str) -> str:
        normalized = os.path.normpath(path).lstrip("/")
        return os.path.join(self._base, normalized)

    def read(self, path: str) -> bytes:
        full = self._full_path(path)
        if not os.path.exists(full):
            raise FileNotFoundError(f"Fichier introuvable : {path}")
        with open(full, "rb") as f:
            return f.read()

    def list_files(self) -> list[str]:
        if not os.path.exists(self._base):
            return []
        result = []
        for root, _, files in os.walk(self._base):
            for name in files:
                result.append(
                    os.path.relpath(os.path.join(root, name), self._base)
                )
        return result


# ── encrypted_storage.py ──────────────────────────────────────────────────────

class EncryptedStorage(ReadWriteStorage):
    """
    Décorateur de ReadWriteStorage qui chiffre à l'écriture et déchiffre
    à la lecture.

    La postcondition read(path) == data écrit est préservée : le
    chiffrement/déchiffrement est transparent pour le code client.
    La postcondition est formellement : read(path) == decrypt(encrypt(data)),
    ce qui est équivalent à data si le chiffrement est correct.

    Raison de changer : changement d'algorithme de chiffrement.
    """

    def __init__(self, inner: ReadWriteStorage, cipher: "CipherInterface"):
        self._inner  = inner
        self._cipher = cipher

    def read(self, path: str) -> bytes:
        encrypted = self._inner.read(path)
        return self._cipher.decrypt(encrypted)

    def write(self, path: str, data: bytes) -> None:
        self._inner.write(path, self._cipher.encrypt(data))

    def delete(self, path: str) -> None:
        self._inner.delete(path)   # délègue directement — pas de chiffrement

    def list_files(self) -> list[str]:
        return self._inner.list_files()


class CipherInterface(ABC):
    @abstractmethod
    def encrypt(self, data: bytes) -> bytes: ...

    @abstractmethod
    def decrypt(self, data: bytes) -> bytes: ...


class XorCipher(CipherInterface):
    """Chiffrement XOR trivial, uniquement pour les tests."""

    def __init__(self, key: int = 0xAB):
        self._key = key

    def encrypt(self, data: bytes) -> bytes:
        return bytes(b ^ self._key for b in data)

    def decrypt(self, data: bytes) -> bytes:
        return self.encrypt(data)  # XOR est son propre inverse


# ── s3_storage.py ─────────────────────────────────────────────────────────────

class S3Storage(ReadWriteStorage):
    """
    Stockage Amazon S3 simulé.

    En production, utiliserait boto3. Ici, on simule avec un dict en mémoire
    pour rendre la classe testable sans infrastructure AWS.

    Respecte intégralement ReadWriteStorage : read, write, delete, list_files
    honorent leurs préconditions et postconditions.
    """

    def __init__(self, bucket: str):
        self._bucket = bucket
        self._store: dict[str, bytes] = {}   # simulacre en mémoire

    def read(self, path: str) -> bytes:
        if path not in self._store:
            raise FileNotFoundError(f"Clé S3 introuvable : s3://{self._bucket}/{path}")
        return self._store[path]

    def write(self, path: str, data: bytes) -> None:
        self._store[path] = data

    def delete(self, path: str) -> None:
        if path not in self._store:
            raise FileNotFoundError(f"Clé S3 introuvable : s3://{self._bucket}/{path}")
        del self._store[path]

    def list_files(self) -> list[str]:
        return list(self._store.keys())
```

### Tests — vérification de la substituabilité

Le test central du LSP est la **fonction générique** : une fonction qui travaille
sur une abstraction doit fonctionner correctement avec *n'importe quelle* implémentation
concrète de cette abstraction.

```python
# ── test_storage.py ───────────────────────────────────────────────────────────

import os
import tempfile
import pytest


# ── Fonctions génériques (code client qui ne connaît que les interfaces) ──────

def store_and_retrieve(storage: ReadWriteStorage, path: str, data: bytes) -> None:
    """
    Preuve de substituabilité : cette fonction fonctionne avec LocalStorage,
    S3Storage, EncryptedStorage — sans aucune modification.
    """
    storage.write(path, data)
    assert storage.read(path) == data
    assert path in storage.list_files()
    storage.delete(path)
    assert path not in storage.list_files()


def read_and_list(storage: ReadableStorage & ListableStorage,
                  path: str, expected: bytes) -> None:
    """Fonctionne avec LocalStorage ET ReadOnlyLocalStorage."""
    assert storage.read(path) == expected
    assert path in storage.list_files()


# ── Fixture partagée ──────────────────────────────────────────────────────────

@pytest.fixture
def temp_dir():
    with tempfile.TemporaryDirectory() as d:
        yield d


# ── Tests de substituabilité ──────────────────────────────────────────────────

class TestReadWriteSubstitutability:
    """
    Chaque ReadWriteStorage doit passer exactement le même test générique.
    C'est la définition opérationnelle du LSP.
    """

    def test_local_storage(self, temp_dir):
        store_and_retrieve(LocalStorage(temp_dir), "notes.txt", b"hello")

    def test_s3_storage(self):
        store_and_retrieve(S3Storage("my-bucket"), "docs/report.pdf", b"%PDF")

    def test_encrypted_storage(self, temp_dir):
        inner  = LocalStorage(temp_dir)
        cipher = XorCipher(0xAB)
        store_and_retrieve(EncryptedStorage(inner, cipher), "secret.bin", b"top secret")

    def test_encrypted_storage_data_is_actually_encrypted(self, temp_dir):
        """Les données sur disque ne doivent pas être en clair."""
        inner  = LocalStorage(temp_dir)
        cipher = XorCipher(0xAB)
        encrypted = EncryptedStorage(inner, cipher)

        encrypted.write("data.bin", b"plaintext")
        raw_on_disk = inner.read("data.bin")

        assert raw_on_disk != b"plaintext"    # les données sont bien chiffrées
        assert encrypted.read("data.bin") == b"plaintext"  # mais transparentes pour le client


class TestReadOnlyStorageConformance:
    """ReadOnlyLocalStorage ne doit pas proposer write/delete,
    mais doit honorer read et list_files aussi fidèlement que LocalStorage."""

    def test_read_returns_correct_bytes(self, temp_dir):
        # On écrit via LocalStorage (qui a le droit d'écrire)...
        writer = LocalStorage(temp_dir)
        writer.write("readme.txt", b"contenu du fichier")

        # ... et on lit via ReadOnlyLocalStorage.
        reader = ReadOnlyLocalStorage(temp_dir)
        assert reader.read("readme.txt") == b"contenu du fichier"

    def test_list_files_returns_existing_files(self, temp_dir):
        writer = LocalStorage(temp_dir)
        writer.write("a.txt", b"a")
        writer.write("b.txt", b"b")

        reader = ReadOnlyLocalStorage(temp_dir)
        files  = reader.list_files()
        assert "a.txt" in files
        assert "b.txt" in files

    def test_read_raises_for_missing_file(self, temp_dir):
        reader = ReadOnlyLocalStorage(temp_dir)
        with pytest.raises(FileNotFoundError):
            reader.read("inexistant.txt")

    def test_does_not_expose_write_or_delete(self):
        """Vérification statique : ReadOnlyLocalStorage n'est pas un WritableStorage."""
        reader = ReadOnlyLocalStorage("/tmp")
        assert not isinstance(reader, WritableStorage)
        assert not hasattr(reader, "write")
        assert not hasattr(reader, "delete")


class TestFileNotFoundBehavior:
    """
    Toutes les implémentations doivent lever FileNotFoundError dans les
    mêmes circonstances — pas d'exception surprise qui briserait le LSP.
    """

    @pytest.mark.parametrize("storage_factory", [
        lambda: S3Storage("bucket"),
    ])
    def test_read_missing_raises_file_not_found(self, storage_factory):
        storage = storage_factory()
        with pytest.raises(FileNotFoundError):
            storage.read("inexistant.txt")

    @pytest.mark.parametrize("storage_factory", [
        lambda: S3Storage("bucket"),
    ])
    def test_delete_missing_raises_file_not_found(self, storage_factory):
        storage = storage_factory()
        with pytest.raises(FileNotFoundError):
            storage.delete("inexistant.txt")
```

### Points clés à retenir

- `ReadOnlyLocalStorage` n'hérite pas de `WritableStorage` : l'impossibilité
  d'écrire est exprimée dans le **type**, pas dans une exception levée à l'exécution.
  Le compilateur (ou le type-checker Python) devient le garant du LSP.
- `EncryptedStorage` est un **décorateur** qui préserve le contrat de
  `ReadWriteStorage` en rendant le chiffrement transparent. La postcondition
  `read(path) == data` reste vraie du point de vue du code client.
- Les tests génériques (`store_and_retrieve`, `read_and_list`) sont la traduction
  directe du LSP en code : si tous les sous-types passent le même test, la
  substituabilité est prouvée.


## Exercice 3 (niveau avancé) — Framework de sérialisation PHP

### Rappel de l'énoncé

Construire une hiérarchie de sérialiseurs (`JsonSerializer`, `XmlSerializer`,
`BinarySerializer`, `WriteOnlySerializer`) en respectant le LSP, en formalisant
les contrats, et en montrant le lien avec l'ISP.

### Étape 1 — Formalisation du contrat de `Serializer`

```
serialize(object $obj): string
  Préconditions  :
    - $obj est une instance d'une classe concrète (non null).
    - La classe de $obj est sérialisable dans le format cible
      (supportsFormat() renvoie true pour ce format).
  Postconditions :
    - Retourne une chaîne non vide.
    - La chaîne est un document valide dans le format cible.
    - L'appel est idempotent : serialize($obj) == serialize($obj) si $obj
      n'a pas changé entre les deux appels.
  Invariant      : ne modifie pas l'état interne de $obj.
  Exceptions     : SerializationException si la sérialisation échoue
                   (classe non supportée, données invalides...).
                   Aucune autre exception ne doit être levée.

deserialize(string $data, string $class): object
  Préconditions  :
    - $data est une chaîne valide dans le format cible (le même que celui
      produit par serialize()).
    - $class est le nom pleinement qualifié d'une classe instanciable.
  Postconditions :
    - Retourne une instance de $class.
    - Les propriétés de l'objet retourné correspondent aux données de $data.
    - Pour tout $obj : deserialize(serialize($obj), get_class($obj)) produit
      un objet égal à $obj (round-trip property).
  Exceptions     : DeserializationException si les données sont invalides.
                   Aucune autre exception (notamment pas NotImplementedError).

supportsFormat(string $format): bool
  Préconditions  : aucune.
  Postconditions : retourne true si et seulement si ce sérialiseur peut
                   traiter le format donné. Jamais d'exception.
```

### Étape 2 — Problème posé par `WriteOnlySerializer`

`WriteOnlySerializer` ne peut pas honorer le contrat de `deserialize`. Deux
options s'offrent à nous, et toutes deux violent le LSP :

**Option A — Lever une exception**

```php
public function deserialize(string $data, string $class): object
{
    throw new \LogicException('Ce sérialiseur ne supporte pas la désérialisation.');
}
```

Violation : la postcondition stipule que `deserialize` retourne toujours une instance
de `$class`. Lever une exception affaiblit cette postcondition. Tout code client qui
suppose qu'un `Serializer` peut désérialiser sera trompé.

**Option B — Retourner silencieusement une valeur arbitraire**

```php
public function deserialize(string $data, string $class): object
{
    return new \stdClass(); // postcondition de round-trip totalement violée
}
```

Violation encore plus grave : le code client croit avoir un objet valide, mais il
obtient un `stdClass` vide.

**Conclusion** : `WriteOnlySerializer` ne *peut pas* être un sous-type de `Serializer`
si `Serializer` garantit la désérialisation. La solution est de décomposer l'interface.

### Étape 3 — Refactorisation respectant le LSP (et l'ISP)

```php
<?php

declare(strict_types=1);

// ── Exceptions ────────────────────────────────────────────────────────────────

class SerializationException extends \RuntimeException {}
class DeserializationException extends \RuntimeException {}


// ── Interfaces ségrégées ──────────────────────────────────────────────────────

/**
 * Contrat de sérialisation pure (objet → chaîne).
 *
 * Postcondition : retourne une chaîne non vide dans le format cible.
 * Invariant     : ne modifie pas $obj.
 * Exception     : SerializationException uniquement.
 */
interface SerializerInterface
{
    public function serialize(object $obj): string;

    public function supportsFormat(string $format): bool;
}

/**
 * Contrat de désérialisation pure (chaîne → objet).
 *
 * Précondition  : $data est valide dans le format cible.
 * Postcondition : retourne une instance de $class.
 *                 Round-trip : deserialize(serialize($obj), get_class($obj)) ≡ $obj
 * Exception     : DeserializationException uniquement.
 */
interface DeserializerInterface
{
    /**
     * @template T of object
     * @param class-string<T> $class
     * @return T
     */
    public function deserialize(string $data, string $class): object;
}

/**
 * Interface composite pour les sérialiseurs bidirectionnels.
 * Hériter de cette interface engage à respecter les deux contrats.
 */
interface BidirectionalSerializerInterface
    extends SerializerInterface, DeserializerInterface {}


// ── Implémentations concrètes ─────────────────────────────────────────────────

final class JsonSerializer implements BidirectionalSerializerInterface
{
    /**
     * Sérialise un objet en JSON.
     * Postcondition : résultat valide selon json_decode().
     */
    public function serialize(object $obj): string
    {
        $json = json_encode($obj, JSON_THROW_ON_ERROR | JSON_UNESCAPED_UNICODE);
        if ($json === false || $json === '') {
            throw new SerializationException("Impossible de sérialiser l'objet en JSON.");
        }
        return $json;
    }

    /**
     * @template T of object
     * @param class-string<T> $class
     * @return T
     */
    public function deserialize(string $data, string $class): object
    {
        try {
            $decoded = json_decode($data, associative: true, flags: JSON_THROW_ON_ERROR);
        } catch (\JsonException $e) {
            throw new DeserializationException("JSON invalide : {$e->getMessage()}", previous: $e);
        }

        if (!class_exists($class)) {
            throw new DeserializationException("Classe inconnue : $class");
        }

        // Hydratation simple via réflexion (production : utiliser un mapper dédié).
        $obj        = new $class();
        $reflection = new \ReflectionClass($obj);
        foreach ($decoded as $property => $value) {
            if ($reflection->hasProperty($property)) {
                $prop = $reflection->getProperty($property);
                $prop->setAccessible(true);
                $prop->setValue($obj, $value);
            }
        }
        return $obj;
    }

    public function supportsFormat(string $format): bool
    {
        return strtolower($format) === 'json';
    }
}


final class XmlSerializer implements BidirectionalSerializerInterface
{
    public function serialize(object $obj): string
    {
        $xml  = new \SimpleXMLElement('<root/>');
        $vars = get_object_vars($obj);

        foreach ($vars as $key => $value) {
            if (is_scalar($value) || $value === null) {
                $xml->addChild($key, (string) ($value ?? ''));
            }
        }

        $result = $xml->asXML();
        if ($result === false) {
            throw new SerializationException("Impossible de produire le XML.");
        }
        return $result;
    }

    public function deserialize(string $data, string $class): object
    {
        try {
            $xml = new \SimpleXMLElement($data);
        } catch (\Exception $e) {
            throw new DeserializationException("XML invalide : {$e->getMessage()}", previous: $e);
        }

        if (!class_exists($class)) {
            throw new DeserializationException("Classe inconnue : $class");
        }

        $obj        = new $class();
        $reflection = new \ReflectionClass($obj);

        foreach ($xml->children() as $child) {
            $name = $child->getName();
            if ($reflection->hasProperty($name)) {
                $prop = $reflection->getProperty($name);
                $prop->setAccessible(true);
                $prop->setValue($obj, (string) $child);
            }
        }
        return $obj;
    }

    public function supportsFormat(string $format): bool
    {
        return strtolower($format) === 'xml';
    }
}


final class BinarySerializer implements BidirectionalSerializerInterface
{
    /**
     * Sérialisation via serialize() natif PHP.
     * Note : en production, préférer un format binaire portable
     * (MessagePack, Protocol Buffers) plutôt que le format interne de PHP.
     */
    public function serialize(object $obj): string
    {
        return base64_encode(serialize($obj));
    }

    public function deserialize(string $data, string $class): object
    {
        $decoded = base64_decode($data, strict: true);
        if ($decoded === false) {
            throw new DeserializationException("Données binaires invalides (base64 corrompu).");
        }

        try {
            $obj = unserialize($decoded, ['allowed_classes' => [$class]]);
        } catch (\Throwable $e) {
            throw new DeserializationException("Désérialisation binaire échouée.", previous: $e);
        }

        if (!($obj instanceof $class)) {
            throw new DeserializationException(
                "L'objet désérialisé n'est pas une instance de $class."
            );
        }
        return $obj;
    }

    public function supportsFormat(string $format): bool
    {
        return strtolower($format) === 'binary';
    }
}


/**
 * WriteOnlySerializer : n'implémente QUE SerializerInterface.
 *
 * Il n'a aucune obligation de désérialiser. Le type-checker empêche
 * statiquement d'utiliser cet objet là où un DeserializerInterface
 * est attendu — le LSP est respecté par construction.
 *
 * Cas d'usage réel : un sérialiseur de logs ou d'audit qui produit
 * une représentation textuelle non réversible (hash, résumé, format
 * propriétaire non parsable).
 */
final class WriteOnlySerializer implements SerializerInterface
{
    public function serialize(object $obj): string
    {
        // Format custom non réversible (ex. : résumé textuel pour l'audit).
        $class = get_class($obj);
        $hash  = substr(md5(serialize($obj)), 0, 8);
        return sprintf('[%s#%s] %s', $class, $hash, json_encode($obj));
    }

    public function supportsFormat(string $format): bool
    {
        return strtolower($format) === 'audit';
    }
}
```

### Orchestrateur respectant le LSP

```php
<?php

declare(strict_types=1);

/**
 * Registre de sérialiseurs.
 * Sépare les collections selon les capacités réelles —
 * aucun code client ne se voit promettre ce qui ne peut pas être tenu.
 */
final class SerializerRegistry
{
    /** @var SerializerInterface[] */
    private array $serializers = [];

    /** @var BidirectionalSerializerInterface[] */
    private array $bidirectional = [];

    public function register(SerializerInterface $serializer): void
    {
        $this->serializers[] = $serializer;
        if ($serializer instanceof BidirectionalSerializerInterface) {
            $this->bidirectional[] = $serializer;
        }
    }

    public function findSerializer(string $format): ?SerializerInterface
    {
        foreach ($this->serializers as $s) {
            if ($s->supportsFormat($format)) {
                return $s;
            }
        }
        return null;
    }

    public function findBidirectional(string $format): ?BidirectionalSerializerInterface
    {
        foreach ($this->bidirectional as $s) {
            if ($s->supportsFormat($format)) {
                return $s;
            }
        }
        return null;
    }
}
```

### Tests PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

// Objet de test simple
final class UserDto
{
    public string $name  = '';
    public string $email = '';
    public int    $age   = 0;
}


/**
 * Test générique du LSP : toutes les implémentations de
 * BidirectionalSerializerInterface doivent passer exactement ce test.
 */
abstract class BidirectionalSerializerTestCase extends TestCase
{
    abstract protected function makeSerializer(): BidirectionalSerializerInterface;
    abstract protected function expectedFormat(): string;

    private function makeUser(): UserDto
    {
        $user        = new UserDto();
        $user->name  = 'Alice Dupont';
        $user->email = 'alice@example.com';
        $user->age   = 30;
        return $user;
    }

    // ── Contrat de SerializerInterface ────────────────────────────────────────

    final public function testSerializeReturnsNonEmptyString(): void
    {
        $result = $this->makeSerializer()->serialize($this->makeUser());
        $this->assertIsString($result);
        $this->assertNotEmpty($result);
    }

    final public function testSupportsFormatReturnsTrueForOwnFormat(): void
    {
        $this->assertTrue(
            $this->makeSerializer()->supportsFormat($this->expectedFormat())
        );
    }

    final public function testSupportsFormatReturnsFalseForUnknownFormat(): void
    {
        $this->assertFalse(
            $this->makeSerializer()->supportsFormat('__unknown__')
        );
    }

    final public function testSerializeIsIdempotent(): void
    {
        $s    = $this->makeSerializer();
        $user = $this->makeUser();
        $this->assertSame($s->serialize($user), $s->serialize($user));
    }

    // ── Contrat de DeserializerInterface (round-trip) ─────────────────────────

    final public function testRoundTripPreservesData(): void
    {
        $s        = $this->makeSerializer();
        $original = $this->makeUser();
        $data     = $s->serialize($original);

        /** @var UserDto $restored */
        $restored = $s->deserialize($data, UserDto::class);

        $this->assertInstanceOf(UserDto::class, $restored);
        $this->assertSame($original->name,  $restored->name);
        $this->assertSame($original->email, $restored->email);
    }

    final public function testDeserializeThrowsOnInvalidData(): void
    {
        $this->expectException(DeserializationException::class);
        $this->makeSerializer()->deserialize('__données_corrompues__', UserDto::class);
    }
}


// Les trois classes concrètes héritent du test générique.
// Si un test échoue pour l'une d'elles, c'est une violation du LSP.

final class JsonSerializerTest extends BidirectionalSerializerTestCase
{
    protected function makeSerializer(): BidirectionalSerializerInterface
    {
        return new JsonSerializer();
    }

    protected function expectedFormat(): string { return 'json'; }
}

final class XmlSerializerTest extends BidirectionalSerializerTestCase
{
    protected function makeSerializer(): BidirectionalSerializerInterface
    {
        return new XmlSerializer();
    }

    protected function expectedFormat(): string { return 'xml'; }
}

final class BinarySerializerTest extends BidirectionalSerializerTestCase
{
    protected function makeSerializer(): BidirectionalSerializerInterface
    {
        return new BinarySerializer();
    }

    protected function expectedFormat(): string { return 'binary'; }
}


// Test spécifique à WriteOnlySerializer (ne passe pas les tests bidirectionnels)
final class WriteOnlySerializerTest extends TestCase
{
    public function testSerializeProducesNonEmptyString(): void
    {
        $s    = new WriteOnlySerializer();
        $user = new UserDto();
        $user->name = 'Bob';
        $this->assertNotEmpty($s->serialize($user));
    }

    public function testIsNotBidirectional(): void
    {
        $s = new WriteOnlySerializer();
        $this->assertNotInstanceOf(BidirectionalSerializerInterface::class, $s);
        $this->assertNotInstanceOf(DeserializerInterface::class, $s);
    }
}


// Test du registre
final class SerializerRegistryTest extends TestCase
{
    public function testFindBidirectionalReturnsNullForAuditFormat(): void
    {
        $registry = new SerializerRegistry();
        $registry->register(new JsonSerializer());
        $registry->register(new WriteOnlySerializer());

        // Le format 'audit' est enregistré mais pas bidirectionnel.
        $this->assertNull($registry->findBidirectional('audit'));
        $this->assertNotNull($registry->findSerializer('audit'));
    }
}
```

### Étape 4 — Lien entre LSP et ISP

Le lien entre les deux principes est direct et réciproque : **une violation du LSP
dans une hiérarchie est souvent le symptôme d'une interface trop large**, et
l'ISP en est le remède.

Voici comment le raisonnement s'enchaîne dans cet exercice :

**1. Observation (violation LSP)**

`WriteOnlySerializer` ne peut pas honorer le contrat de `deserialize`. Sa présence
dans une hiérarchie qui promet la désérialisation force à lever une exception ou
à retourner une valeur incohérente — deux formes d'affaiblissement de postcondition.

**2. Diagnostic (interface trop large)**

L'interface `Serializer` d'origine regroupe deux capacités conceptuellement
distinctes : la *sérialisation* (objet → texte) et la *désérialisation* (texte →
objet). Regrouper ces deux capacités dans une seule interface impose à tout
sous-type de les supporter toutes les deux — même quand cela n'a pas de sens.
C'est exactement ce que l'ISP interdit.

**3. Remède (ségrégation des interfaces)**

On sépare `SerializerInterface` et `DeserializerInterface`. `WriteOnlySerializer`
n'implémente que la première. `BidirectionalSerializerInterface` combine les deux
pour les cas où les deux capacités sont disponibles.

**4. Conséquence (LSP rétabli)**

Après la ségrégation, chaque interface a un contrat que *tous* ses sous-types peuvent
honorer sans affaiblissement. Il n'y a plus de `NotImplementedError` surprise ni de
postcondition violée. La substituabilité est garantie par construction.

La formulation synthétique de cette relation :

> L'ISP empêche de *promettre* ce qu'une classe ne peut pas *tenir*.
> Le LSP exige que ce qui est *promis* soit *tenu*.
> Respecter l'ISP est une condition nécessaire (mais pas suffisante) au respect du LSP.

La condition « pas suffisante » s'illustre par `EncryptedStorage` : même avec une
interface correctement ségrégée, une implémentation défaillante (déchiffrement qui
retourne des données corrompues) violerait le LSP sans qu'aucune interface ne soit
en cause. Le LSP exige aussi la correction de l'implémentation, pas seulement la
bonne conception des interfaces.

---

## Synthèse des deux exercices LSP

| Aspect | Exercice 2 (stockage Python) | Exercice 3 (sérialisation PHP) |
|---|---|---|
| Violation identifiée | `ReadOnlyLocalStorage.write()` lèverait `NotImplementedError` | `WriteOnlySerializer.deserialize()` ne peut pas honorer la postcondition |
| Mécanisme de violation | Affaiblissement de postcondition | Affaiblissement de postcondition |
| Remède | Ségrégation en `ReadableStorage` / `WritableStorage` / `ListableStorage` | Ségrégation en `SerializerInterface` / `DeserializerInterface` |
| Preuve de conformité | Test générique `store_and_retrieve` passé par tous les `ReadWriteStorage` | Classe de test abstraite `BidirectionalSerializerTestCase` héritée par chaque implémentation |
| Lien avec l'ISP | Interface trop large → impossible à implémenter complètement | Même diagnostic, même remède |

Dans les deux exercices, la démarche est identique :

1. **Formaliser le contrat** avant d'écrire le code (pré/postconditions).
2. **Identifier la classe qui ne peut pas tenir le contrat** complet.
3. **Segmenter l'interface** pour que chaque fragment soit honorable par tous ses
   sous-types.
4. **Écrire un test générique** qui prouve la substituabilité de façon automatisée.

L'étape 4 est la plus puissante : un test générique qui tourne en vert pour toutes
les implémentations est une démonstration formelle du LSP, bien plus convaincante
qu'un raisonnement purement théorique.
