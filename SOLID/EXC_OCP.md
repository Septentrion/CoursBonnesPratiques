# OCP — Corrigés des exercices niveaux 2 et 3

Comme pour le SRP, ces corrigés proposent une solution de référence parmi d'autres
architectures valides. L'accent est mis sur le **raisonnement** qui conduit à chaque
choix de conception, pas uniquement sur le code final.

---

## Exercice 2 (niveau intermédiaire) — Système de validation de formulaire extensible en Python

### Rappel de l'énoncé

Concevoir un `FormValidator` qui :
- applique une liste de règles sur un dictionnaire de données ;
- peut recevoir n'importe quelle nouvelle règle sans être modifié ;
- expose au moins quatre règles concrètes (`RequiredRule`, `MinLengthRule`,
  `EmailRule`, `NumericRule`…) ;
- prouve sa extensibilité par un test de composition.

### Analyse OCP

La violation classique serait un `FormValidator` avec un `if/elif` par type de
règle. Chaque nouvel ajout forcerait à rouvrir la classe — exactement ce que l'OCP
interdit. La solution canonique : définir un **contrat de règle** (une interface
abstraite) et laisser `FormValidator` itérer dessus sans savoir ce qu'il y a dedans.

`FormValidator` devient ainsi **fermé à la modification** (son code ne change plus)
et **ouvert à l'extension** (on ajoute des règles en créant de nouvelles classes).

### Implémentation

```python
# ── validation_rules.py ──────────────────────────────────────────────────────

from __future__ import annotations

import re
from abc import ABC, abstractmethod
from dataclasses import dataclass


# ── Contrat commun à toutes les règles ───────────────────────────────────────

@dataclass(frozen=True)
class ValidationError:
    field: str
    message: str


class ValidationRule(ABC):
    """
    Contrat que toute règle doit respecter.
    FormValidator ne connaît que cette interface — jamais les classes concrètes.
    """

    @abstractmethod
    def validate(self, field: str, value: object) -> ValidationError | None:
        """Retourne une erreur si la règle est violée, None sinon."""
        ...


# ── Règles concrètes ─────────────────────────────────────────────────────────

class RequiredRule(ValidationRule):
    """Le champ ne doit pas être absent, None ou chaîne vide."""

    def validate(self, field: str, value: object) -> ValidationError | None:
        if value is None or (isinstance(value, str) and not value.strip()):
            return ValidationError(field, f"Le champ « {field} » est obligatoire.")
        return None


class MinLengthRule(ValidationRule):
    """La valeur (chaîne) doit comporter au moins `min_length` caractères."""

    def __init__(self, min_length: int):
        self._min = min_length

    def validate(self, field: str, value: object) -> ValidationError | None:
        if not isinstance(value, str):
            return None  # règle inapplicable à un non-string : on laisse passer
        if len(value) < self._min:
            return ValidationError(
                field,
                f"« {field} » doit contenir au moins {self._min} caractères "
                f"(actuellement {len(value)})."
            )
        return None


class MaxLengthRule(ValidationRule):
    """La valeur ne doit pas dépasser `max_length` caractères."""

    def __init__(self, max_length: int):
        self._max = max_length

    def validate(self, field: str, value: object) -> ValidationError | None:
        if isinstance(value, str) and len(value) > self._max:
            return ValidationError(
                field,
                f"« {field} » ne doit pas dépasser {self._max} caractères."
            )
        return None


class EmailRule(ValidationRule):
    """La valeur doit ressembler à une adresse e-mail valide."""

    _PATTERN = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

    def validate(self, field: str, value: object) -> ValidationError | None:
        if isinstance(value, str) and not self._PATTERN.match(value):
            return ValidationError(field, f"« {field} » n'est pas une adresse e-mail valide.")
        return None


class NumericRule(ValidationRule):
    """La valeur doit être un entier ou un flottant (ou une chaîne convertible)."""

    def validate(self, field: str, value: object) -> ValidationError | None:
        if isinstance(value, (int, float)):
            return None
        if isinstance(value, str):
            try:
                float(value)
                return None
            except ValueError:
                pass
        return ValidationError(field, f"« {field} » doit être une valeur numérique.")


class RangeRule(ValidationRule):
    """La valeur numérique doit être comprise entre `min_val` et `max_val`."""

    def __init__(self, min_val: float, max_val: float):
        self._min = min_val
        self._max = max_val

    def validate(self, field: str, value: object) -> ValidationError | None:
        try:
            num = float(value)  # type: ignore[arg-type]
        except (TypeError, ValueError):
            return None  # NumericRule s'en chargera
        if not (self._min <= num <= self._max):
            return ValidationError(
                field,
                f"« {field} » doit être compris entre {self._min} et {self._max}."
            )
        return None


# ── Règles ajoutées SANS modifier FormValidator ni les règles ci-dessus ──────

class CreditCardRule(ValidationRule):
    """
    Vérifie qu'un numéro de carte bancaire est syntaxiquement valide
    via l'algorithme de Luhn.
    Ajout pur : zéro modification du code existant.
    """

    def validate(self, field: str, value: object) -> ValidationError | None:
        if not isinstance(value, str):
            return None
        digits = value.replace(" ", "").replace("-", "")
        if not digits.isdigit() or len(digits) < 13:
            return ValidationError(field, f"« {field} » : numéro de carte invalide.")
        if not self._luhn(digits):
            return ValidationError(field, f"« {field} » : la somme de contrôle Luhn échoue.")
        return None

    @staticmethod
    def _luhn(number: str) -> bool:
        total = 0
        for i, digit in enumerate(reversed(number)):
            n = int(digit)
            if i % 2 == 1:
                n *= 2
                if n > 9:
                    n -= 9
            total += n
        return total % 10 == 0


class PhoneNumberRule(ValidationRule):
    """Accepte les numéros de téléphone français (formats E.164 ou local)."""

    _PATTERN = re.compile(r"^(?:\+33|0033|0)[1-9](?:[\s.\-]?\d{2}){4}$")

    def validate(self, field: str, value: object) -> ValidationError | None:
        if isinstance(value, str) and not self._PATTERN.match(value.strip()):
            return ValidationError(field, f"« {field} » : numéro de téléphone invalide.")
        return None


# ── validator.py ─────────────────────────────────────────────────────────────

class FieldRules:
    """Association entre un champ et la liste des règles qui s'y appliquent."""

    def __init__(self, field: str, *rules: ValidationRule):
        self.field = field
        self.rules = list(rules)


class FormValidator:
    """
    Orchestre la validation d'un formulaire.

    OUVERT à l'extension  : on injecte autant de FieldRules que nécessaire.
    FERME à la modification : cette classe ne change jamais quand on ajoute une règle.

    Elle ignore totalement ce que font les règles concrètes — elle sait seulement
    qu'elles respectent l'interface ValidationRule.
    """

    def __init__(self, *field_rules: FieldRules):
        self._field_rules = list(field_rules)

    def validate(self, data: dict[str, object]) -> list[ValidationError]:
        errors: list[ValidationError] = []
        for fr in self._field_rules:
            value = data.get(fr.field)
            for rule in fr.rules:
                error = rule.validate(fr.field, value)
                if error is not None:
                    errors.append(error)
                    break  # on s'arrête à la première erreur par champ (fail-fast)
        return errors

    def is_valid(self, data: dict[str, object]) -> bool:
        return len(self.validate(data)) == 0
```

### Tests de composition

```python
# ── test_form_validator.py ────────────────────────────────────────────────────

import pytest


# ── Utilitaire ───────────────────────────────────────────────────────────────

def make_registration_validator() -> FormValidator:
    """
    Formulaire d'inscription : username, email, age, password.
    La composition des règles se fait à l'extérieur de FormValidator.
    """
    return FormValidator(
        FieldRules("username",
                   RequiredRule(),
                   MinLengthRule(3),
                   MaxLengthRule(20)),
        FieldRules("email",
                   RequiredRule(),
                   EmailRule()),
        FieldRules("age",
                   RequiredRule(),
                   NumericRule(),
                   RangeRule(13, 120)),
        FieldRules("password",
                   RequiredRule(),
                   MinLengthRule(8)),
    )


# ── Tests de chaque règle isolément ──────────────────────────────────────────

class TestRequiredRule:
    def test_none_is_invalid(self):
        assert RequiredRule().validate("x", None) is not None

    def test_empty_string_is_invalid(self):
        assert RequiredRule().validate("x", "  ") is not None

    def test_non_empty_is_valid(self):
        assert RequiredRule().validate("x", "hello") is None


class TestEmailRule:
    @pytest.mark.parametrize("email", ["alice@example.com", "a+b@x.io"])
    def test_valid_emails(self, email):
        assert EmailRule().validate("email", email) is None

    @pytest.mark.parametrize("email", ["notanemail", "missing@dot", "@nodomain.com"])
    def test_invalid_emails(self, email):
        assert EmailRule().validate("email", email) is not None


class TestRangeRule:
    def test_within_range(self):
        assert RangeRule(0, 100).validate("score", 50) is None

    def test_below_range(self):
        assert RangeRule(18, 99).validate("age", 10) is not None

    def test_above_range(self):
        assert RangeRule(18, 99).validate("age", 150) is not None


class TestCreditCardRule:
    def test_valid_visa_number(self):
        # 4111111111111111 est un numéro de test Visa qui passe Luhn
        assert CreditCardRule().validate("card", "4111111111111111") is None

    def test_invalid_number(self):
        assert CreditCardRule().validate("card", "1234567890123456") is not None

    def test_non_numeric_rejected(self):
        assert CreditCardRule().validate("card", "abcd-efgh-ijkl-mnop") is not None


# ── Tests de composition ──────────────────────────────────────────────────────

class TestFormValidator:
    def test_valid_form_returns_no_errors(self):
        validator = make_registration_validator()
        data = {
            "username": "alice42",
            "email":    "alice@example.com",
            "age":      "25",
            "password": "s3cr3tP@ss",
        }
        assert validator.is_valid(data)

    def test_missing_required_fields(self):
        validator = make_registration_validator()
        errors = validator.validate({"username": "", "email": None, "age": None, "password": None})
        fields_in_error = {e.field for e in errors}
        assert {"username", "email", "age", "password"} == fields_in_error

    def test_invalid_email_and_age(self):
        validator = make_registration_validator()
        errors = validator.validate({
            "username": "bob",
            "email":    "not-an-email",
            "age":      "twelve",
            "password": "longpassword",
        })
        fields = {e.field for e in errors}
        assert "email" in fields
        assert "age" in fields

    def test_age_out_of_range(self):
        validator = make_registration_validator()
        errors = validator.validate({
            "username": "carol",
            "email":    "carol@example.com",
            "age":      "200",
            "password": "password123",
        })
        assert any(e.field == "age" for e in errors)

    def test_adding_credit_card_rule_does_not_modify_validator(self):
        """
        Preuve d'OCP : on étend la validation avec CreditCardRule
        sans toucher à FormValidator.
        """
        payment_validator = FormValidator(
            FieldRules("card_number",
                       RequiredRule(),
                       CreditCardRule()),
            FieldRules("phone",
                       RequiredRule(),
                       PhoneNumberRule()),
        )
        valid_data = {
            "card_number": "4111111111111111",
            "phone":       "06 12 34 56 78",
        }
        assert payment_validator.is_valid(valid_data)

    def test_phone_number_validation(self):
        rule = PhoneNumberRule()
        assert rule.validate("phone", "+33612345678") is None
        assert rule.validate("phone", "0033612345678") is None
        assert rule.validate("phone", "notaphone") is not None
```

### Points clés à retenir

- `FormValidator` n'a jamais besoin d'être modifié : il dépend uniquement de
  `ValidationRule`, une abstraction stable.
- Ajouter `CreditCardRule` ou `PhoneNumberRule` se fait en créant une nouvelle classe.
  Le test `test_adding_credit_card_rule_does_not_modify_validator` le prouve
  explicitement.
- La composition des règles par champ (`FieldRules`) est déclarée à l'extérieur
  du validateur — c'est le **Composition Root**, seul endroit légitime qui connaît
  les classes concrètes.
- Le comportement *fail-fast* (arrêt à la première erreur par champ) est une
  décision centralisée dans `FormValidator` ; changer ce comportement ne touche
  que cette classe.

---

## Exercice 3 (niveau avancé) — Pipeline de traitement d'images en PHP

### Rappel de l'énoncé

1. Définir une interface `ImageFilter` et implémenter quatre filtres concrets.
2. Implémenter `ImagePipeline` pour enchaîner les filtres dans l'ordre de construction.
3. Ajouter `AiEnhancementFilter` sans modifier `ImagePipeline` ni aucun filtre existant.
4. Discuter les limites de l'OCP.

### Analyse OCP

Un pipeline est l'illustration parfaite de l'OCP : l'orchestrateur (`ImagePipeline`)
est stable par construction — il ne fait qu'itérer sur une liste de filtres. Chaque
filtre est une unité d'extension indépendante. La seule chose qui « s'ouvre » lors
d'un ajout est la liste des filtres fournie à la construction — et cette liste vit
dans le Composition Root, pas dans le pipeline lui-même.

### Implémentation

```php
<?php

declare(strict_types=1);

// ── ImageData.php ─────────────────────────────────────────────────────────────

/**
 * Représente une image en cours de traitement.
 * Objet immuable : chaque filtre retourne une nouvelle instance.
 */
final class ImageData
{
    public function __construct(
        public readonly string $path,        // chemin du fichier source
        public readonly int    $width,       // largeur en pixels
        public readonly int    $height,      // hauteur en pixels
        public readonly string $colorMode,   // 'rgb' | 'grayscale'
        public readonly array  $metadata,    // données arbitraires portées entre filtres
    ) {}

    public function withDimensions(int $width, int $height): self
    {
        return new self($this->path, $width, $height, $this->colorMode, $this->metadata);
    }

    public function withColorMode(string $mode): self
    {
        return new self($this->path, $this->width, $this->height, $mode, $this->metadata);
    }

    public function withMeta(string $key, mixed $value): self
    {
        return new self(
            $this->path, $this->width, $this->height, $this->colorMode,
            array_merge($this->metadata, [$key => $value])
        );
    }
}


// ── ImageFilter.php ───────────────────────────────────────────────────────────

/**
 * Contrat unique que tout filtre doit respecter.
 * ImagePipeline ne dépend que de cette interface — jamais des filtres concrets.
 */
interface ImageFilter
{
    /**
     * Applique le filtre à l'image et retourne le résultat.
     * L'image d'entrée n'est pas modifiée (immutabilité recommandée).
     */
    public function apply(ImageData $image): ImageData;

    /** Nom lisible du filtre, utile pour les logs et le débogage. */
    public function name(): string;
}


// ── Filtres concrets ──────────────────────────────────────────────────────────

/**
 * Redimensionne l'image selon une largeur cible en conservant les proportions.
 */
final class ResizeFilter implements ImageFilter
{
    public function __construct(private readonly int $targetWidth) {}

    public function apply(ImageData $image): ImageData
    {
        if ($image->width === 0) {
            return $image;
        }
        $ratio  = $this->targetWidth / $image->width;
        $newH   = (int) round($image->height * $ratio);

        // En production : appel GD ou Imagick ici.
        // $gd = imagescale($src, $this->targetWidth, $newH);

        return $image
            ->withDimensions($this->targetWidth, $newH)
            ->withMeta('resized_to', "{$this->targetWidth}x{$newH}");
    }

    public function name(): string
    {
        return "ResizeFilter(width={$this->targetWidth})";
    }
}


/**
 * Convertit l'image en niveaux de gris.
 */
final class GrayscaleFilter implements ImageFilter
{
    public function apply(ImageData $image): ImageData
    {
        // En production : imagefilter($gd, IMG_FILTER_GRAYSCALE)
        return $image
            ->withColorMode('grayscale')
            ->withMeta('grayscale', true);
    }

    public function name(): string
    {
        return 'GrayscaleFilter';
    }
}


/**
 * Applique un filigrane textuel en bas à droite de l'image.
 */
final class WatermarkFilter implements ImageFilter
{
    public function __construct(
        private readonly string $text,
        private readonly float  $opacity = 0.5,
    ) {}

    public function apply(ImageData $image): ImageData
    {
        // En production : imagettftext() avec positionnement dynamique.
        return $image->withMeta('watermark', [
            'text'    => $this->text,
            'opacity' => $this->opacity,
            'x'       => (int) ($image->width  * 0.75),
            'y'       => (int) ($image->height * 0.95),
        ]);
    }

    public function name(): string
    {
        return "WatermarkFilter(text=\"{$this->text}\", opacity={$this->opacity})";
    }
}


/**
 * Compresse l'image JPEG au niveau de qualité demandé.
 */
final class CompressionFilter implements ImageFilter
{
    public function __construct(
        private readonly int $quality = 85  // 0 (max compression) – 100 (pas de perte)
    ) {
        if ($quality < 0 || $quality > 100) {
            throw new \InvalidArgumentException('La qualité doit être entre 0 et 100.');
        }
    }

    public function apply(ImageData $image): ImageData
    {
        // En production : imagejpeg($gd, $path, $this->quality)
        return $image->withMeta('compression_quality', $this->quality);
    }

    public function name(): string
    {
        return "CompressionFilter(quality={$this->quality})";
    }
}


// ── ImagePipeline.php ─────────────────────────────────────────────────────────

/**
 * Orchestre l'exécution séquentielle d'une liste de filtres.
 *
 * FERME à la modification : cette classe ne changera jamais pour accueillir
 *   un nouveau filtre.
 * OUVERT à l'extension   : on lui injecte n'importe quelle liste de ImageFilter.
 *
 * Elle ignore totalement ce que font les filtres — elle sait seulement
 * qu'ils respectent l'interface ImageFilter.
 */
final class ImagePipeline
{
    /** @var ImageFilter[] */
    private array $filters;

    public function __construct(ImageFilter ...$filters)
    {
        $this->filters = $filters;
    }

    /**
     * Exécute chaque filtre dans l'ordre et retourne l'image finale.
     * Chaque étape reçoit le résultat de l'étape précédente.
     */
    public function process(ImageData $image): ImageData
    {
        $current = $image;
        foreach ($this->filters as $filter) {
            $current = $filter->apply($current);
        }
        return $current;
    }

    /**
     * Retourne un nouveau pipeline avec un filtre ajouté en fin de chaîne.
     * Utile pour construire des pipelines de manière fluente et immuable.
     */
    public function pipe(ImageFilter $filter): self
    {
        return new self(...$this->filters, ...[$filter]);
    }

    /** @return string[] */
    public function describe(): array
    {
        return array_map(fn(ImageFilter $f) => $f->name(), $this->filters);
    }
}
```

### Ajout de `AiEnhancementFilter` sans aucune modification

```php
<?php

declare(strict_types=1);

/**
 * Filtre ajouté a posteriori — aucun fichier existant n'est modifié.
 *
 * Appelle une API externe simulée pour améliorer la qualité d'image par IA
 * (super-résolution, débruitage, amélioration de la netteté).
 */
final class AiEnhancementFilter implements ImageFilter
{
    public function __construct(
        private readonly AiImageApiClientInterface $apiClient,
        private readonly string                    $model = 'enhance-v2',
    ) {}

    public function apply(ImageData $image): ImageData
    {
        $result = $this->apiClient->enhance(
            imagePath: $image->path,
            model:     $this->model,
        );

        return $image
            ->withDimensions($result['width'], $result['height'])
            ->withMeta('ai_enhanced', true)
            ->withMeta('ai_model',    $this->model)
            ->withMeta('ai_score',    $result['quality_score']);
    }

    public function name(): string
    {
        return "AiEnhancementFilter(model={$this->model})";
    }
}


// ── Client de l'API IA (abstraction + implémentation simulée) ─────────────────

interface AiImageApiClientInterface
{
    /**
     * @return array{width: int, height: int, quality_score: float}
     */
    public function enhance(string $imagePath, string $model): array;
}

/** Implémentation de production (appel HTTP réel). */
final class HttpAiImageApiClient implements AiImageApiClientInterface
{
    public function __construct(
        private readonly string $baseUrl,
        private readonly string $apiKey,
    ) {}

    public function enhance(string $imagePath, string $model): array
    {
        // En production : appel cURL / Guzzle vers l'API.
        // Ici, on simule simplement un doublement de résolution.
        throw new \RuntimeException('Appel HTTP non implémenté dans cet exemple.');
    }
}

/** Doublure de test. */
final class FakeAiImageApiClient implements AiImageApiClientInterface
{
    public function enhance(string $imagePath, string $model): array
    {
        return [
            'width'         => 2048,
            'height'        => 1536,
            'quality_score' => 0.95,
        ];
    }
}
```

### Tests PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

/**
 * Image de départ utilisée dans tous les tests.
 */
function makeTestImage(): ImageData
{
    return new ImageData(
        path:      '/tmp/photo.jpg',
        width:     1200,
        height:    800,
        colorMode: 'rgb',
        metadata:  [],
    );
}


final class ResizeFilterTest extends TestCase
{
    public function testResizeKeepsAspectRatio(): void
    {
        $image  = makeTestImage();                       // 1200 x 800
        $filter = new ResizeFilter(600);
        $result = $filter->apply($image);

        $this->assertSame(600, $result->width);
        $this->assertSame(400, $result->height);        // ratio 3:2 préservé
    }

    public function testMetaIsRecorded(): void
    {
        $result = (new ResizeFilter(800))->apply(makeTestImage());
        $this->assertArrayHasKey('resized_to', $result->metadata);
    }
}


final class GrayscaleFilterTest extends TestCase
{
    public function testColorModeChanges(): void
    {
        $result = (new GrayscaleFilter())->apply(makeTestImage());
        $this->assertSame('grayscale', $result->colorMode);
    }
}


final class WatermarkFilterTest extends TestCase
{
    public function testWatermarkMetaIsStored(): void
    {
        $result = (new WatermarkFilter('© Mon Studio', 0.7))->apply(makeTestImage());
        $this->assertSame('© Mon Studio', $result->metadata['watermark']['text']);
        $this->assertSame(0.7, $result->metadata['watermark']['opacity']);
    }
}


final class CompressionFilterTest extends TestCase
{
    public function testQualityMetaIsStored(): void
    {
        $result = (new CompressionFilter(75))->apply(makeTestImage());
        $this->assertSame(75, $result->metadata['compression_quality']);
    }

    public function testInvalidQualityThrows(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        new CompressionFilter(150);
    }
}


final class ImagePipelineTest extends TestCase
{
    /**
     * Pipeline complet : redimensionnement → niveaux de gris → filigrane → compression.
     * Vérifie que toutes les transformations sont appliquées dans l'ordre.
     */
    public function testPipelineAppliesAllFiltersInOrder(): void
    {
        $pipeline = new ImagePipeline(
            new ResizeFilter(800),
            new GrayscaleFilter(),
            new WatermarkFilter('© Test'),
            new CompressionFilter(80),
        );

        $result = $pipeline->process(makeTestImage());

        $this->assertSame(800, $result->width);
        $this->assertSame('grayscale', $result->colorMode);
        $this->assertArrayHasKey('watermark', $result->metadata);
        $this->assertSame(80, $result->metadata['compression_quality']);
    }

    /**
     * Preuve d'OCP : AiEnhancementFilter s'intègre sans modifier ImagePipeline.
     */
    public function testAiFilterIntegratesWithoutModifyingPipeline(): void
    {
        $fakeClient = new FakeAiImageApiClient();

        // On réutilise le pipeline existant et on ajoute le filtre IA via pipe().
        $basePipeline = new ImagePipeline(
            new ResizeFilter(800),
            new CompressionFilter(85),
        );

        $enhancedPipeline = $basePipeline->pipe(new AiEnhancementFilter($fakeClient));

        $result = $enhancedPipeline->process(makeTestImage());

        $this->assertTrue($result->metadata['ai_enhanced']);
        $this->assertSame('enhance-v2', $result->metadata['ai_model']);
        $this->assertSame(2048, $result->width);  // résolution améliorée par l'IA
    }

    public function testDescribeReturnsFilterNames(): void
    {
        $pipeline = new ImagePipeline(
            new ResizeFilter(600),
            new GrayscaleFilter(),
        );

        $names = $pipeline->describe();

        $this->assertSame('ResizeFilter(width=600)', $names[0]);
        $this->assertSame('GrayscaleFilter',         $names[1]);
    }

    public function testEmptyPipelineReturnsUnchangedImage(): void
    {
        $image  = makeTestImage();
        $result = (new ImagePipeline())->process($image);

        $this->assertSame($image->width,     $result->width);
        $this->assertSame($image->colorMode, $result->colorMode);
        $this->assertEmpty($result->metadata);
    }
}
```

### Point 4 — Limites de l'OCP : quand la modification est inévitable ou préférable

L'OCP est une heuristique, pas une loi. Voici les situations où s'y tenir coûte plus
qu'il ne rapporte.

#### Cas 1 — La découverte de l'axe d'extension vrai

Quand on écrit le code pour la première fois, on ne sait pas toujours *comment* il
sera étendu. Appliquer l'OCP prématurément crée des abstractions arbitraires qui
figent le mauvais axe de variation. Martin Fowler appelle cela l'over-engineering
préventif.

Exemple concret : notre `ImageFilter` fonctionne parce que tous les filtres partagent
la même signature `apply(ImageData): ImageData`. Si on découvrait plus tard qu'un
filtre nécessite une configuration asynchrone ou un contexte de transaction, l'interface
devrait changer — et tous les filtres avec elle. L'abstraction est solide parce que
l'axe d'extension est connu et stable, pas parce que l'OCP a été appliqué à l'aveugle.

**Règle pratique** : n'abstraire que les axes de variation qu'on a déjà vus changer
au moins une fois (règle des « deux instances »). La première, on la code en dur ;
la deuxième, on abstrait.

#### Cas 2 — Un bug dans l'abstraction de base

Si `ImageFilter` a un contrat défaillant (par exemple, elle ne précise pas ce qui
se passe si `ImageData.path` est invalide), corriger ce contrat oblige à modifier
l'interface *et* toutes ses implémentations. C'est une modification en cascade
inévitable — et c'est juste : l'abstraction elle-même était incorrecte.

**Conséquence** : le coût d'une mauvaise abstraction est proportionnel au nombre
d'implémentations. Plus on a de filtres, plus corriger l'interface est douloureux.
Cela justifie de prendre le temps de bien concevoir l'interface dès le départ et
d'écrire un contrat précis (pré/postconditions) plutôt que de se précipiter.

#### Cas 3 — L'optimisation croisée

Certains filtres doivent coopérer pour être efficaces. Par exemple, un filtre de
redimensionnement suivi d'un filtre de super-résolution IA pourrait économiser
beaucoup de temps de calcul en détectant cette combinaison et en la remplaçant
par une seule opération. Ce type d'optimisation *croisée* est impossible à ajouter
sans modifier soit `ImagePipeline`, soit l'interface des filtres.

Les solutions classiques (pattern Visitor, règles de réécriture de pipeline,
optimiseurs de graphes de calcul comme ceux de PyTorch ou XLA) ajoutent une
couche de complexité significative. Si la performance n'est pas un problème, ce
coût n'est pas justifié.

**Règle pratique** : l'OCP est plus facile à respecter quand les unités d'extension
sont *indépendantes*. Dès qu'elles doivent interagir entre elles, l'abstraction
devient plus complexe et la tentation de modifier `ImagePipeline` est forte.

#### Cas 4 — La lisibilité l'emporte sur l'extensibilité

Pour une petite application avec deux ou trois cas connus et stables, un simple
`switch/match` est plus lisible et plus facile à déboguer qu'une hiérarchie de
classes. L'OCP a un coût : plus de fichiers, plus d'indirections, plus de
complexité cognitive.

```php
// Ceci est parfois la meilleure solution :
function applyFilter(ImageData $img, string $type): ImageData
{
    return match($type) {
        'resize'    => (new ResizeFilter(800))->apply($img),
        'grayscale' => (new GrayscaleFilter())->apply($img),
        default     => throw new \InvalidArgumentException("Filtre inconnu : $type"),
    };
}
```

Si l'application ne grandit pas, ce code est excellent. L'OCP ne doit pas devenir
un dogme qui pousse à créer des abstractions sans valeur ajoutée réelle.

---

## Synthèse des deux exercices OCP

| Aspect | Exercice 2 (FormValidator) | Exercice 3 (ImagePipeline) |
|---|---|---|
| Mécanisme d'extension | Injection de règles à la construction | Injection de filtres à la construction |
| Point stable (fermé) | `FormValidator.validate()` | `ImagePipeline.process()` |
| Point d'extension (ouvert) | Nouvelles classes `ValidationRule` | Nouvelles classes `ImageFilter` |
| Composition Root | Site d'appel (`make_registration_validator()`) | Site d'appel (`new ImagePipeline(...)`) |
| Testabilité | Chaque règle testée isolément | Chaque filtre testé isolément |
| Preuve OCP | `CreditCardRule` ajoutée sans toucher `FormValidator` | `AiEnhancementFilter` ajouté sans toucher `ImagePipeline` |

Dans les deux cas, la structure est identique :

1. Une **abstraction stable** définit le contrat (`ValidationRule`, `ImageFilter`).
2. Un **orchestrateur fermé** itère sur une liste de ces abstractions.
3. Les **extensions concrètes** n'ont pas conscience les unes des autres.
4. La **composition** est déléguée au Composition Root, seul endroit qui connaît
   les classes concrètes.

Ce motif — souvent appelé **Strategy** ou **Chain of Responsibility** selon le
contexte — est la principale application pratique de l'OCP dans le code orienté objet.
