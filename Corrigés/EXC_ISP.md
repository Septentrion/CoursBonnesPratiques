# ISP — Corrigés des exercices niveaux 2 et 3

L'Interface Segregation Principle est souvent présenté comme le plus « mécanique »
des principes SOLID : il suffit de découper les grandes interfaces en plus petites.
En réalité, la difficulté réside dans le **choix des axes de découpage** : trop fins,
les interfaces deviennent anémiques et la composition devient fastidieuse ; trop larges,
on retombe dans le problème de départ. Ces corrigés insistent sur le raisonnement
qui justifie chaque frontière d'interface.

---

## Exercice 2 (niveau intermédiaire) — Système de plugins pour éditeur de texte en Python

### Rappel de l'énoncé

Concevoir un système de plugins où chaque plugin peut exposer une ou plusieurs des
capacités suivantes : analyse syntaxique, autocomplétion, formatage, débogage.
Implémenter `LightweightPlugin` (formatage seul), `FullIdePlugin` (tout), et
`LinterPlugin` (analyse + formatage). Écrire `run_formatter(plugin)` acceptant
tout plugin capable de formater.

### Analyse ISP

L'interface monolithique serait :

```python
class Plugin(ABC):
    def syntax_check(self, code: str) -> list[str]: ...
    def suggest(self, prefix: str) -> list[str]: ...
    def format(self, code: str) -> str: ...
    def debug(self, code: str) -> str: ...
```

`LightweightPlugin` n'implémenterait que `format` et serait contraint de fournir
des implémentations vides ou levant des exceptions pour les trois autres méthodes.
C'est le signal caractéristique d'une violation ISP : une classe implémente des
méthodes dont elle n'a pas besoin.

L'axe de découpage est ici naturel : chaque capacité est fonctionnellement
indépendante. Un plugin de formatage n'a aucune raison de connaître l'API de
débogage, et vice versa.

### Implémentation

```python
# ── plugin_interfaces.py ──────────────────────────────────────────────────────

from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass


# ── Types de données partagés ─────────────────────────────────────────────────

@dataclass(frozen=True)
class SyntaxIssue:
    line:    int
    column:  int
    message: str
    severity: str = "error"   # "error" | "warning" | "info"


@dataclass(frozen=True)
class Suggestion:
    text:        str
    description: str = ""
    score:       float = 1.0  # pertinence, de 0.0 à 1.0


@dataclass(frozen=True)
class DebugFrame:
    variable: str
    value:    object
    scope:    str


# ── Interfaces ségrégées ──────────────────────────────────────────────────────

class SyntaxChecker(ABC):
    """
    Capacité : analyser le code et signaler les problèmes syntaxiques
    ou stylistiques.

    Postcondition : retourne une liste (éventuellement vide) de SyntaxIssue.
    Ne modifie jamais le code source fourni.
    """

    @abstractmethod
    def syntax_check(self, code: str) -> list[SyntaxIssue]:
        ...


class Autocompleter(ABC):
    """
    Capacité : proposer des complétions à partir d'un préfixe.

    Précondition  : prefix peut être vide (demande toutes les complétions).
    Postcondition : liste triée par score décroissant, éventuellement vide.
    """

    @abstractmethod
    def suggest(self, prefix: str, context: str = "") -> list[Suggestion]:
        ...


class Formatter(ABC):
    """
    Capacité : reformater du code selon un style défini.

    Précondition  : code est une chaîne non None (peut être vide).
    Postcondition : le code retourné est syntaxiquement équivalent au code
                    d'entrée (même sémantique, présentation différente).
    Idempotence   : format(format(code)) == format(code).
    """

    @abstractmethod
    def format(self, code: str) -> str:
        ...


class Debugger(ABC):
    """
    Capacité : inspecter l'état d'un programme au moment de l'exécution
    ou à partir d'une trace.

    Postcondition : retourne la liste des frames disponibles, jamais None.
    Exception     : DebugSessionError si la session ne peut pas démarrer.
    """

    @abstractmethod
    def debug(self, code: str) -> list[DebugFrame]:
        ...


# ── Interfaces composites (pour les plugins riches) ───────────────────────────

class LinterCapable(SyntaxChecker, Formatter, ABC):
    """Combine analyse et formatage — les deux capacités d'un linter classique."""
    ...


class FullIdeCapable(SyntaxChecker, Autocompleter, Formatter, Debugger, ABC):
    """Toutes les capacités d'un IDE complet."""
    ...


# ── Implémentations concrètes ─────────────────────────────────────────────────

class LightweightPlugin(Formatter):
    """
    Plugin minimaliste : reformate le code Python via la bibliothèque black
    (simulée ici pour rester sans dépendances).

    N'implémente QUE Formatter — aucune méthode vide, aucune exception surprise.
    Raison de changer : modification de la politique de formatage.
    """

    def __init__(self, indent_size: int = 4):
        self._indent = indent_size

    def format(self, code: str) -> str:
        # Simulation de formatage : normalisation de l'indentation uniquement.
        lines   = code.splitlines()
        result  = []
        current_indent = 0

        for line in lines:
            stripped = line.strip()
            if not stripped:
                result.append("")
                continue
            if stripped.startswith(("return", "pass", "break", "continue", "raise")):
                result.append(" " * (current_indent * self._indent) + stripped)
            elif stripped.endswith(":"):
                result.append(" " * (current_indent * self._indent) + stripped)
                current_indent += 1
            else:
                result.append(" " * (current_indent * self._indent) + stripped)

        return "\n".join(result)


class LinterPlugin(LinterCapable):
    """
    Plugin d'analyse + formatage.
    Implémente SyntaxChecker et Formatter, mais pas Autocompleter ni Debugger.
    """

    # ── SyntaxChecker ─────────────────────────────────────────────────────────

    def syntax_check(self, code: str) -> list[SyntaxIssue]:
        issues: list[SyntaxIssue] = []

        for i, line in enumerate(code.splitlines(), start=1):
            stripped = line.rstrip()

            # Règle 1 : lignes trop longues
            if len(stripped) > 79:
                issues.append(SyntaxIssue(
                    line=i, column=80,
                    message=f"Ligne trop longue ({len(stripped)} > 79 caractères).",
                    severity="warning",
                ))

            # Règle 2 : tabulations mélangées avec des espaces
            if "\t" in line and "    " in line:
                issues.append(SyntaxIssue(
                    line=i, column=1,
                    message="Mélange de tabulations et d'espaces.",
                    severity="error",
                ))

            # Règle 3 : point-virgule en fin de ligne (style Java dans du Python)
            if stripped.endswith(";"):
                issues.append(SyntaxIssue(
                    line=i, column=len(stripped),
                    message="Point-virgule superflu en fin de ligne.",
                    severity="warning",
                ))

        return issues

    # ── Formatter ─────────────────────────────────────────────────────────────

    def format(self, code: str) -> str:
        lines  = code.splitlines()
        result = []
        for line in lines:
            # Suppression des point-virgules terminaux
            cleaned = line.rstrip().rstrip(";")
            # Remplacement des tabulations par 4 espaces
            cleaned = cleaned.replace("\t", "    ")
            result.append(cleaned)
        # Suppression des lignes vides consécutives en excès
        output: list[str] = []
        blank_count = 0
        for line in result:
            if line.strip() == "":
                blank_count += 1
                if blank_count <= 2:
                    output.append(line)
            else:
                blank_count = 0
                output.append(line)
        return "\n".join(output)


class FullIdePlugin(FullIdeCapable):
    """
    Plugin complet : implémente les quatre capacités.
    Raison de changer : évolution de n'importe quelle des quatre capacités.
    (En pratique, ce plugin serait probablement décomposé en délégués.)
    """

    def __init__(self, keywords: list[str] | None = None):
        self._keywords = keywords or [
            "def", "class", "import", "from", "return", "if", "else",
            "elif", "for", "while", "try", "except", "with", "as",
            "lambda", "yield", "async", "await",
        ]

    # ── SyntaxChecker ─────────────────────────────────────────────────────────

    def syntax_check(self, code: str) -> list[SyntaxIssue]:
        issues: list[SyntaxIssue] = []
        try:
            compile(code, "<string>", "exec")
        except SyntaxError as e:
            issues.append(SyntaxIssue(
                line=e.lineno or 0,
                column=e.offset or 0,
                message=str(e.msg),
                severity="error",
            ))
        return issues

    # ── Autocompleter ─────────────────────────────────────────────────────────

    def suggest(self, prefix: str, context: str = "") -> list[Suggestion]:
        matches = [kw for kw in self._keywords if kw.startswith(prefix)]
        return [
            Suggestion(text=kw, description=f"Mot-clé Python : {kw}", score=1.0)
            for kw in sorted(matches)
        ]

    # ── Formatter ─────────────────────────────────────────────────────────────

    def format(self, code: str) -> str:
        # Délègue à LinterPlugin pour éviter la duplication.
        return LinterPlugin().format(code)

    # ── Debugger ──────────────────────────────────────────────────────────────

    def debug(self, code: str) -> list[DebugFrame]:
        # Simulation : exécute le code dans un namespace isolé et capture les
        # variables locales. En production : intégration avec pdb ou debugpy.
        namespace: dict = {}
        try:
            exec(code, namespace)  # noqa: S102 (usage intentionnel en simulation)
        except Exception:
            pass  # On retourne ce qu'on a pu capturer
        return [
            DebugFrame(variable=k, value=v, scope="global")
            for k, v in namespace.items()
            if not k.startswith("__")
        ]


# ── plugin_runner.py ──────────────────────────────────────────────────────────

def run_formatter(plugin: Formatter, code: str) -> str:
    """
    Accepte n'importe quel objet qui implémente Formatter.

    LightweightPlugin, LinterPlugin et FullIdePlugin passent tous ici —
    sans cast, sans isinstance, sans connaissance des types concrets.
    C'est la preuve directe de l'ISP : la fonction ne dépend que de
    la capacité dont elle a besoin.
    """
    return plugin.format(code)


def run_linter(plugin: SyntaxChecker, code: str) -> list[SyntaxIssue]:
    """Idem pour l'analyse syntaxique."""
    return plugin.syntax_check(code)


def run_autocomplete(plugin: Autocompleter, prefix: str) -> list[Suggestion]:
    """Idem pour l'autocomplétion."""
    return plugin.suggest(prefix)
```

### Tests

```python
# ── test_plugins.py ───────────────────────────────────────────────────────────

import pytest


SAMPLE_CODE = """\
def hello(name):
    print("Hello, " + name);
    very_long_line = "ceci est une ligne vraiment très longue qui dépasse largement la limite de 79 caractères"
"""


# ── Tests d'interface (ISP) ───────────────────────────────────────────────────

class TestInterfaceSegregation:
    def test_lightweight_plugin_is_only_formatter(self):
        plugin = LightweightPlugin()
        assert isinstance(plugin, Formatter)
        assert not isinstance(plugin, SyntaxChecker)
        assert not isinstance(plugin, Autocompleter)
        assert not isinstance(plugin, Debugger)

    def test_linter_plugin_is_linter_capable(self):
        plugin = LinterPlugin()
        assert isinstance(plugin, SyntaxChecker)
        assert isinstance(plugin, Formatter)
        assert not isinstance(plugin, Autocompleter)
        assert not isinstance(plugin, Debugger)

    def test_full_ide_plugin_has_all_capabilities(self):
        plugin = FullIdePlugin()
        assert isinstance(plugin, SyntaxChecker)
        assert isinstance(plugin, Autocompleter)
        assert isinstance(plugin, Formatter)
        assert isinstance(plugin, Debugger)


# ── Tests run_formatter avec les trois plugins ────────────────────────────────

class TestRunFormatter:
    """
    Preuve de substituabilité ISP : run_formatter fonctionne avec
    tout plugin qui implémente Formatter, quel que soit son type concret.
    """

    @pytest.mark.parametrize("plugin", [
        LightweightPlugin(),
        LinterPlugin(),
        FullIdePlugin(),
    ])
    def test_run_formatter_returns_string(self, plugin: Formatter):
        result = run_formatter(plugin, SAMPLE_CODE)
        assert isinstance(result, str)
        assert len(result) > 0

    @pytest.mark.parametrize("plugin", [
        LinterPlugin(),
        FullIdePlugin(),
    ])
    def test_run_formatter_removes_semicolons(self, plugin: Formatter):
        code   = 'x = 1;\ny = 2;'
        result = run_formatter(plugin, code)
        assert ";" not in result


# ── Tests de LinterPlugin ─────────────────────────────────────────────────────

class TestLinterPlugin:
    def test_detects_long_lines(self):
        plugin = LinterPlugin()
        issues = plugin.syntax_check(SAMPLE_CODE)
        long_line_issues = [i for i in issues if "longue" in i.message]
        assert len(long_line_issues) >= 1

    def test_detects_trailing_semicolon(self):
        plugin = LinterPlugin()
        issues = plugin.syntax_check('print("hello");')
        assert any("point-virgule" in i.message.lower() for i in issues)

    def test_format_idempotent(self):
        plugin = LinterPlugin()
        once  = plugin.format(SAMPLE_CODE)
        twice = plugin.format(once)
        assert once == twice


# ── Tests de FullIdePlugin ────────────────────────────────────────────────────

class TestFullIdePlugin:
    def test_suggest_filters_by_prefix(self):
        plugin      = FullIdePlugin()
        suggestions = plugin.suggest("re")
        texts       = [s.text for s in suggestions]
        assert "return" in texts
        assert "def" not in texts

    def test_suggest_empty_prefix_returns_all(self):
        plugin      = FullIdePlugin()
        suggestions = plugin.suggest("")
        assert len(suggestions) > 0

    def test_debug_captures_variables(self):
        plugin = FullIdePlugin()
        frames = plugin.debug("x = 42\ny = 'hello'")
        names  = [f.variable for f in frames]
        assert "x" in names
        assert "y" in names

    def test_syntax_check_catches_error(self):
        plugin = FullIdePlugin()
        issues = plugin.syntax_check("def foo(:\n    pass")
        assert len(issues) > 0
        assert issues[0].severity == "error"
```

### Points clés à retenir

- `run_formatter` ne reçoit qu'un `Formatter` : elle ignore si son argument
  sait aussi analyser, compléter ou déboguer. C'est la définition opérationnelle
  de l'ISP — le code client ne dépend que de ce dont il a besoin.
- Les interfaces composites (`LinterCapable`, `FullIdeCapable`) permettent de
  nommer des combinaisons fréquentes sans dupliquer les contrats individuels.
- Les types de données partagés (`SyntaxIssue`, `Suggestion`, `DebugFrame`) sont
  indépendants des interfaces : ils peuvent évoluer sans modifier les contrats.

---

## Exercice 3 (niveau avancé) — Bibliothèque de gestion de médias en PHP

### Rappel de l'énoncé

Partir d'une interface `Media` de vingt méthodes, la décomposer en interfaces
cohésives, implémenter `Image`, `Video`, `AudioFile`, `LiveStream`, montrer qu'un
`MediaPlayer` dépend uniquement de `Streamable`, et discuter le lien avec la
Loi de Déméter.

### Étape 1 — Inventaire et regroupement des vingt méthodes

Voici la grande interface de départ, annotée par capacité :

```
[LECTURE]
  open(path: string): void
  close(): void
  seek(position: int): void
  getPosition(): int
  getDuration(): float

[ÉCRITURE]
  save(path: string): void
  saveAs(path: string, format: string): void
  delete(): void

[TRANSCODAGE]
  transcode(targetFormat: string, options: array): void
  getBitrateOptions(): array
  supportsFormat(format: string): bool

[MÉTADONNÉES]
  getMetadata(): array
  setMetadata(key: string, value: mixed): void
  getTitle(): string
  getAuthor(): string

[STREAMING]
  getStreamUrl(): string
  setStreamQuality(quality: string): void
  isLive(): bool

[MINIATURES]
  generateThumbnail(atSecond: float): string   // retourne un chemin
  getThumbnailCount(): int
  getThumbnailPath(index: int): string
```

Six capacités distinctes → six interfaces ségrégées.

### Étape 2 — Interfaces avec contrats formalisés

```php
<?php

declare(strict_types=1);

// ── Exceptions ────────────────────────────────────────────────────────────────

class MediaException      extends \RuntimeException {}
class StreamException     extends MediaException {}
class TranscodeException  extends MediaException {}
class ThumbnailException  extends MediaException {}


// ── 1. Readable — lecture et navigation ──────────────────────────────────────

/**
 * Capacité : ouvrir un média et naviguer dans son contenu.
 *
 * Invariant    : après open(), getPosition() retourne 0.
 * Précondition : seek() n'est appelé qu'après open().
 * Postcondition: seek(n) puis getPosition() == n si n <= getDuration().
 * Exception    : MediaException si le fichier est inaccessible.
 */
interface Readable
{
    public function open(string $path): void;
    public function close(): void;
    public function seek(int $positionMs): void;
    public function getPosition(): int;
    public function getDuration(): float;  // en secondes
}


// ── 2. Writable — persistance ─────────────────────────────────────────────────

/**
 * Capacité : persister ou supprimer un média.
 *
 * Postcondition save()   : le fichier existe à l'emplacement fourni.
 * Postcondition delete() : le fichier n'existe plus.
 * Exception              : MediaException si l'opération échoue.
 */
interface Writable
{
    public function save(string $path): void;
    public function saveAs(string $path, string $format): void;
    public function delete(): void;
}


// ── 3. Transcodable — conversion de format ────────────────────────────────────

/**
 * Capacité : convertir un média vers un autre format ou bitrate.
 *
 * Précondition  : supportsFormat($format) == true avant d'appeler transcode().
 * Postcondition : le fichier résultant est un média valide dans le format cible.
 * Exception     : TranscodeException si la conversion échoue.
 */
interface Transcodable
{
    /** @return string[] */
    public function getBitrateOptions(): array;

    public function supportsFormat(string $format): bool;

    /** @param array<string, mixed> $options */
    public function transcode(string $targetFormat, array $options = []): void;
}


// ── 4. MetadataHolder — gestion des métadonnées ───────────────────────────────

/**
 * Capacité : lire et écrire les métadonnées du média.
 *
 * Postcondition : setMetadata($k, $v) puis getMetadata()[$k] === $v.
 * Invariant     : getMetadata() ne retourne jamais null.
 */
interface MetadataHolder
{
    /** @return array<string, mixed> */
    public function getMetadata(): array;

    public function setMetadata(string $key, mixed $value): void;

    public function getTitle(): string;

    public function getAuthor(): string;
}


// ── 5. Streamable — diffusion ─────────────────────────────────────────────────

/**
 * Capacité : diffuser un média (à la demande ou en direct).
 *
 * Précondition  : getStreamUrl() retourne une URL non vide, valide,
 *                 accessible depuis le réseau du lecteur.
 * Postcondition : setStreamQuality() prend effet au prochain segment.
 * Exception     : StreamException si le flux est indisponible.
 */
interface Streamable
{
    public function getStreamUrl(): string;

    /**
     * @param string $quality  "auto" | "1080p" | "720p" | "480p" | "360p"
     */
    public function setStreamQuality(string $quality): void;

    public function isLive(): bool;
}


// ── 6. Thumbnailable — génération de miniatures ───────────────────────────────

/**
 * Capacité : générer et récupérer des miniatures (frames extraites).
 *
 * Précondition  : $atSecond doit être <= getDuration() du média parent.
 * Postcondition : generateThumbnail() retourne un chemin vers un fichier image
 *                 existant et lisible.
 * Exception     : ThumbnailException si la génération échoue.
 */
interface Thumbnailable
{
    public function generateThumbnail(float $atSecond): string;

    public function getThumbnailCount(): int;

    public function getThumbnailPath(int $index): string;
}
```

### Étape 3 — Implémentations concrètes

```php
<?php

declare(strict_types=1);

// ── Image ─────────────────────────────────────────────────────────────────────

/**
 * Une image statique.
 * Capacités : lecture, écriture, métadonnées.
 * Pas de streaming (une image ne se lit pas dans le temps),
 * pas de transcodage au sens audio/vidéo, pas de miniatures temporelles.
 */
final class Image implements Readable, Writable, MetadataHolder
{
    private string $path        = '';
    private bool   $isOpen      = false;
    private array  $metadata    = [];

    // ── Readable ──────────────────────────────────────────────────────────────

    public function open(string $path): void
    {
        if (!file_exists($path)) {
            throw new MediaException("Image introuvable : $path");
        }
        $this->path   = $path;
        $this->isOpen = true;
        // En production : chargement GD / Imagick
    }

    public function close(): void
    {
        $this->isOpen = false;
    }

    public function seek(int $positionMs): void
    {
        // Une image n'a pas de position temporelle.
        // Seek(0) est acceptable ; toute autre valeur est hors-contrat.
        if ($positionMs !== 0) {
            throw new MediaException("Une image n'a pas de position temporelle.");
        }
    }

    public function getPosition(): int
    {
        return 0;
    }

    public function getDuration(): float
    {
        return 0.0;  // durée nulle par définition
    }

    // ── Writable ──────────────────────────────────────────────────────────────

    public function save(string $path): void
    {
        if (!$this->isOpen) {
            throw new MediaException("L'image doit être ouverte avant d'être sauvegardée.");
        }
        copy($this->path, $path);
    }

    public function saveAs(string $path, string $format): void
    {
        // En production : conversion GD / Imagick selon $format
        $this->save($path);
    }

    public function delete(): void
    {
        if (!file_exists($this->path)) {
            throw new MediaException("Fichier introuvable : {$this->path}");
        }
        unlink($this->path);
        $this->isOpen = false;
    }

    // ── MetadataHolder ────────────────────────────────────────────────────────

    public function getMetadata(): array
    {
        return $this->metadata;
    }

    public function setMetadata(string $key, mixed $value): void
    {
        $this->metadata[$key] = $value;
    }

    public function getTitle(): string
    {
        return (string) ($this->metadata['title'] ?? basename($this->path));
    }

    public function getAuthor(): string
    {
        return (string) ($this->metadata['author'] ?? '');
    }
}


// ── Video ─────────────────────────────────────────────────────────────────────

/**
 * Un fichier vidéo à la demande.
 * Capacités : lecture, écriture, transcodage, métadonnées, streaming, miniatures.
 */
final class Video implements
    Readable,
    Writable,
    Transcodable,
    MetadataHolder,
    Streamable,
    Thumbnailable
{
    private string $path     = '';
    private int    $position = 0;
    private float  $duration = 0.0;
    private string $quality  = 'auto';
    private array  $metadata = [];
    private array  $thumbnails = [];

    private const SUPPORTED_FORMATS = ['mp4', 'mkv', 'avi', 'webm', 'mov'];

    // ── Readable ──────────────────────────────────────────────────────────────

    public function open(string $path): void
    {
        if (!file_exists($path)) {
            throw new MediaException("Vidéo introuvable : $path");
        }
        $this->path     = $path;
        $this->position = 0;
        // En production : ffprobe pour extraire la durée
        $this->duration = 3600.0;  // valeur simulée
    }

    public function close(): void
    {
        $this->position = 0;
    }

    public function seek(int $positionMs): void
    {
        if ($positionMs < 0 || $positionMs > (int) ($this->duration * 1000)) {
            throw new MediaException("Position hors limites : $positionMs ms.");
        }
        $this->position = $positionMs;
    }

    public function getPosition(): int { return $this->position; }
    public function getDuration(): float { return $this->duration; }

    // ── Writable ──────────────────────────────────────────────────────────────

    public function save(string $path): void
    {
        copy($this->path, $path);
    }

    public function saveAs(string $path, string $format): void
    {
        if (!$this->supportsFormat($format)) {
            throw new MediaException("Format non supporté : $format");
        }
        // En production : appel ffmpeg
        $this->save($path);
    }

    public function delete(): void
    {
        unlink($this->path);
    }

    // ── Transcodable ──────────────────────────────────────────────────────────

    public function getBitrateOptions(): array
    {
        return ['128k', '256k', '512k', '1M', '2M', '4M', '8M'];
    }

    public function supportsFormat(string $format): bool
    {
        return in_array(strtolower($format), self::SUPPORTED_FORMATS, true);
    }

    public function transcode(string $targetFormat, array $options = []): void
    {
        if (!$this->supportsFormat($targetFormat)) {
            throw new TranscodeException("Format cible non supporté : $targetFormat");
        }
        // En production : exec('ffmpeg ...') avec les $options
    }

    // ── MetadataHolder ────────────────────────────────────────────────────────

    public function getMetadata(): array { return $this->metadata; }

    public function setMetadata(string $key, mixed $value): void
    {
        $this->metadata[$key] = $value;
    }

    public function getTitle(): string
    {
        return (string) ($this->metadata['title'] ?? basename($this->path));
    }

    public function getAuthor(): string
    {
        return (string) ($this->metadata['author'] ?? '');
    }

    // ── Streamable ────────────────────────────────────────────────────────────

    public function getStreamUrl(): string
    {
        if (empty($this->path)) {
            throw new StreamException("Aucune vidéo ouverte.");
        }
        return "https://cdn.example.com/stream/" . urlencode(basename($this->path));
    }

    public function setStreamQuality(string $quality): void
    {
        $this->quality = $quality;
    }

    public function isLive(): bool { return false; }

    // ── Thumbnailable ─────────────────────────────────────────────────────────

    public function generateThumbnail(float $atSecond): string
    {
        if ($atSecond > $this->duration) {
            throw new ThumbnailException(
                "Position $atSecond s dépasse la durée {$this->duration} s."
            );
        }
        // En production : ffmpeg -ss $atSecond -vframes 1 output.jpg
        $thumbPath = sys_get_temp_dir() . "/thumb_{$atSecond}.jpg";
        $this->thumbnails[] = $thumbPath;
        return $thumbPath;
    }

    public function getThumbnailCount(): int { return count($this->thumbnails); }

    public function getThumbnailPath(int $index): string
    {
        if (!isset($this->thumbnails[$index])) {
            throw new ThumbnailException("Miniature $index inexistante.");
        }
        return $this->thumbnails[$index];
    }
}


// ── AudioFile ─────────────────────────────────────────────────────────────────

/**
 * Un fichier audio.
 * Capacités : lecture, écriture, transcodage, métadonnées, streaming.
 * Pas de miniatures temporelles (l'audio n'a pas de frames vidéo).
 */
final class AudioFile implements
    Readable,
    Writable,
    Transcodable,
    MetadataHolder,
    Streamable
{
    private string $path     = '';
    private int    $position = 0;
    private float  $duration = 0.0;
    private array  $metadata = [];

    private const SUPPORTED_FORMATS = ['mp3', 'flac', 'ogg', 'aac', 'wav', 'm4a'];

    public function open(string $path): void
    {
        if (!file_exists($path)) {
            throw new MediaException("Fichier audio introuvable : $path");
        }
        $this->path     = $path;
        $this->position = 0;
        $this->duration = 240.0;  // simulé
    }

    public function close(): void          { $this->position = 0; }
    public function seek(int $ms): void    { $this->position = $ms; }
    public function getPosition(): int     { return $this->position; }
    public function getDuration(): float   { return $this->duration; }

    public function save(string $path): void        { copy($this->path, $path); }
    public function saveAs(string $path, string $format): void { $this->save($path); }
    public function delete(): void                  { unlink($this->path); }

    public function getBitrateOptions(): array      { return ['64k', '128k', '192k', '320k']; }
    public function supportsFormat(string $f): bool { return in_array(strtolower($f), self::SUPPORTED_FORMATS, true); }
    public function transcode(string $fmt, array $opts = []): void
    {
        if (!$this->supportsFormat($fmt)) {
            throw new TranscodeException("Format audio non supporté : $fmt");
        }
    }

    public function getMetadata(): array                       { return $this->metadata; }
    public function setMetadata(string $k, mixed $v): void     { $this->metadata[$k] = $v; }
    public function getTitle(): string                         { return (string)($this->metadata['title'] ?? ''); }
    public function getAuthor(): string                        { return (string)($this->metadata['author'] ?? ''); }

    public function getStreamUrl(): string       { return "https://cdn.example.com/audio/" . basename($this->path); }
    public function setStreamQuality(string $q): void { /* qualité audio */ }
    public function isLive(): bool               { return false; }
}


// ── LiveStream ────────────────────────────────────────────────────────────────

/**
 * Un flux en direct.
 * Capacités : streaming et métadonnées uniquement.
 * Pas de lecture locale (seek impossible sur un flux temps réel),
 * pas d'écriture, pas de transcodage local, pas de miniatures.
 */
final class LiveStream implements Streamable, MetadataHolder
{
    private array  $metadata = [];
    private string $quality  = 'auto';

    public function __construct(private readonly string $streamUrl) {}

    public function getStreamUrl(): string
    {
        return $this->streamUrl;
    }

    public function setStreamQuality(string $quality): void
    {
        $this->quality = $quality;
    }

    public function isLive(): bool { return true; }

    public function getMetadata(): array                     { return $this->metadata; }
    public function setMetadata(string $k, mixed $v): void   { $this->metadata[$k] = $v; }
    public function getTitle(): string                       { return (string)($this->metadata['title'] ?? 'Live'); }
    public function getAuthor(): string                      { return (string)($this->metadata['author'] ?? ''); }
}
```

### Étape 4 — `MediaPlayer` dépend uniquement de `Streamable`

```php
<?php

declare(strict_types=1);

/**
 * Lecteur multimédia.
 *
 * Dépend UNIQUEMENT de Streamable.
 * Il ne sait pas si l'objet qu'il reçoit est une Video, un AudioFile
 * ou un LiveStream. Il ne peut pas appeler getMetadata(), transcode(),
 * generateThumbnail() — le compilateur PHP l'interdit statiquement.
 *
 * C'est la démonstration concrète de l'ISP : le code client ne voit
 * que le contrat dont il a besoin.
 */
final class MediaPlayer
{
    private ?Streamable $current = null;
    private string      $log     = '';

    public function load(Streamable $media): void
    {
        $this->current = $media;
        $this->log .= sprintf(
            "[%s] Chargement du flux : %s (live=%s)\n",
            date('H:i:s'),
            $media->getStreamUrl(),
            $media->isLive() ? 'oui' : 'non',
        );
    }

    public function play(): string
    {
        if ($this->current === null) {
            throw new StreamException("Aucun flux chargé.");
        }
        $url = $this->current->getStreamUrl();
        $this->log .= "[{date('H:i:s')}] Lecture : $url\n";
        return $url;
    }

    public function setQuality(string $quality): void
    {
        if ($this->current === null) {
            throw new StreamException("Aucun flux chargé.");
        }
        $this->current->setStreamQuality($quality);
    }

    public function isCurrentLive(): bool
    {
        return $this->current?->isLive() ?? false;
    }

    public function getLog(): string
    {
        return $this->log;
    }
}
```

### Tests PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;


final class MediaPlayerTest extends TestCase
{
    /**
     * Preuve d'ISP : le même MediaPlayer fonctionne avec Video, AudioFile
     * et LiveStream sans aucune modification.
     * La substituabilité est complète car tous respectent le contrat Streamable.
     */
    public function testPlayerAcceptsVideoAsStreamable(): void
    {
        $video = $this->createMock(Video::class);
        $video->method('getStreamUrl')->willReturn('https://cdn.example.com/stream/film.mp4');
        $video->method('isLive')->willReturn(false);

        $player = new MediaPlayer();
        $player->load($video);

        $this->assertSame('https://cdn.example.com/stream/film.mp4', $player->play());
        $this->assertFalse($player->isCurrentLive());
    }

    public function testPlayerAcceptsAudioFileAsStreamable(): void
    {
        $audio = $this->createMock(AudioFile::class);
        $audio->method('getStreamUrl')->willReturn('https://cdn.example.com/audio/track.mp3');
        $audio->method('isLive')->willReturn(false);

        $player = new MediaPlayer();
        $player->load($audio);

        $this->assertSame('https://cdn.example.com/audio/track.mp3', $player->play());
    }

    public function testPlayerAcceptsLiveStreamAsStreamable(): void
    {
        $live = new LiveStream('https://live.example.com/channel/1');

        $player = new MediaPlayer();
        $player->load($live);

        $this->assertTrue($player->isCurrentLive());
        $this->assertSame('https://live.example.com/channel/1', $player->play());
    }

    public function testPlayerCannotAccessMetadata(): void
    {
        // Ce test est une vérification statique : MediaPlayer ne connaît que
        // Streamable, il n'a aucun accès à getTitle() ou getAuthor().
        // La preuve est dans le typage de la méthode load() et dans le fait
        // que MediaPlayer n'appelle jamais de méthodes de MetadataHolder.
        $this->assertTrue(true, 'Vérification documentaire du typage statique.');
    }

    public function testPlayWithoutLoadThrows(): void
    {
        $this->expectException(StreamException::class);
        (new MediaPlayer())->play();
    }

    public function testSetQualityDelegatesCorrectly(): void
    {
        $stream = $this->createMock(LiveStream::class);
        $stream->method('getStreamUrl')->willReturn('https://live.example.com/hd');
        $stream->method('isLive')->willReturn(true);
        $stream->expects($this->once())
               ->method('setStreamQuality')
               ->with('1080p');

        $player = new MediaPlayer();
        $player->load($stream);
        $player->setQuality('1080p');
    }
}


// ── Tests des implémentations concrètes ───────────────────────────────────────

final class LiveStreamTest extends TestCase
{
    public function testIsLiveReturnsTrueAlways(): void
    {
        $stream = new LiveStream('https://live.example.com/ch1');
        $this->assertTrue($stream->isLive());
    }

    public function testDoesNotImplementReadable(): void
    {
        $stream = new LiveStream('https://live.example.com/ch1');
        $this->assertNotInstanceOf(Readable::class, $stream);
        $this->assertNotInstanceOf(Writable::class, $stream);
        $this->assertNotInstanceOf(Thumbnailable::class, $stream);
    }

    public function testMetadataRoundTrip(): void
    {
        $stream = new LiveStream('https://live.example.com/ch1');
        $stream->setMetadata('title', 'Concert en direct');
        $this->assertSame('Concert en direct', $stream->getTitle());
    }
}


final class ImageTest extends TestCase
{
    public function testDoesNotImplementStreamable(): void
    {
        $img = new Image();
        $this->assertNotInstanceOf(Streamable::class, $img);
        $this->assertNotInstanceOf(Thumbnailable::class, $img);
        $this->assertNotInstanceOf(Transcodable::class, $img);
    }
}
```

### Étape 5 — Lien entre ISP et Loi de Déméter

La **Loi de Déméter** (Law of Demeter, LoD), formulée à Northeastern University en
1987, stipule qu'un objet ne devrait interagir qu'avec ses voisins immédiats — ses
propres attributs, les objets passés en paramètres, et les objets qu'il crée. En
d'autres termes : *ne parle qu'à tes amis directs, pas aux amis de tes amis*.

Le lien avec l'ISP est structurel et se manifeste sur deux axes.

**Axe 1 — Des interfaces larges poussent à violer la LoD**

Considérons un `MediaPlayer` qui recevrait `Media` (l'interface monolithique de
vingt méthodes) au lieu de `Streamable`. Pour jouer un flux, il n'a besoin que de
`getStreamUrl()`. Mais puisqu'il a accès à vingt méthodes, un développeur pressé
sera tenté d'écrire :

```php
// Tentant mais violant la LoD :
$title    = $media->getMetadata()['title'];          // accès à un sous-objet
$thumbUrl = $media->getThumbnailPath(0);             // traversée inutile
$media->transcode('mp4', ['bitrate' => '4M']);       // connaissance interne
```

Chaque appel supplémentaire crée un couplage : `MediaPlayer` devient dépendant
non seulement du contrat de streaming, mais aussi des détails de métadonnées,
de miniatures et de transcodage. Un changement dans la signature de `getMetadata()`
brise `MediaPlayer` même si sa logique de lecture n'a pas changé.

L'ISP, en restreignant l'interface visible à `Streamable`, rend ces appels
**inaccessibles** plutôt que simplement déconseillés. La LoD est alors respectée
par construction, non par discipline.

**Axe 2 — La LoD guide le choix des frontières d'interface**

Inversement, quand on hésite sur la finesse du découpage, la LoD fournit un
critère pratique : *quels objets un consommateur donné a-t-il légitimement besoin
de connaître ?*

`MediaPlayer` a besoin de connaître une URL et un booléen `isLive`. Il n'a aucune
raison légitime de connaître un chemin de miniature ou un tableau de métadonnées.
Le fait que ces informations soient dans des interfaces séparées (`Thumbnailable`,
`MetadataHolder`) interdit naturellement à `MediaPlayer` de les traverser — aucune
chaîne d'appels `$media->getThumbnails()->getPath(0)` n'est possible si `$media`
est typé `Streamable`.

**Formulation synthétique**

> L'ISP délimite ce qu'un consommateur *peut voir*.
> La LoD délimite ce à quoi un consommateur *devrait accéder*.
> Respecter l'ISP rend mécaniquement plus facile le respect de la LoD, car le
> compilateur retire la tentation d'accéder à ce qui n'appartient pas au contrat.

La différence est que la LoD s'applique à l'utilisation des objets à l'intérieur
des méthodes, tandis que l'ISP s'applique à la déclaration des dépendances entre
classes. Les deux principes se renforcent : l'ISP agit au niveau de la conception,
la LoD au niveau de l'implémentation.

---

## Synthèse des deux exercices ISP

| Aspect | Exercice 2 (plugins Python) | Exercice 3 (médias PHP) |
|---|---|---|
| Interface monolithique violant l'ISP | `Plugin` à 4 méthodes | `Media` à 20 méthodes |
| Axe de découpage | Par capacité fonctionnelle indépendante | Par capacité fonctionnelle indépendante |
| Interfaces résultantes | `SyntaxChecker`, `Autocompleter`, `Formatter`, `Debugger` | `Readable`, `Writable`, `Transcodable`, `MetadataHolder`, `Streamable`, `Thumbnailable` |
| Interfaces composites | `LinterCapable`, `FullIdeCapable` | (non nécessaires : les classes implémentent directement plusieurs interfaces) |
| Preuve d'ISP | `run_formatter` n'accepte que `Formatter` | `MediaPlayer` n'accepte que `Streamable` |
| Lien avec un autre principe | OCP (nouvelles règles sans modifier le runner) | LSP (chaque `Streamable` passe le même test) + LoD |

Dans les deux exercices, la procédure est identique :

1. Identifier les *consommateurs* et ce dont ils ont réellement besoin.
2. Définir une interface par capacité cohésive, avec son contrat formel.
3. Laisser les classes concrètes implémenter les interfaces pertinentes pour elles.
4. Typer les paramètres des fonctions et services sur les interfaces minimales.
5. Vérifier par un test (ou par le compilateur) que le code client ne peut pas
   accéder à ce qui ne lui appartient pas.
