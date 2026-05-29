
#### Corrigé — Exercice A (String Calculator, 8 étapes)

```python
# ── string_calculator.py ──────────────────────────────────────────────────────

import re


class NegativeNumberException(ValueError):
    def __init__(self, negatives: list[int]):
        self.negatives = negatives
        super().__init__(f"Nombres négatifs non autorisés : {negatives}")


def add(numbers: str) -> int:
    """
    Calcule la somme des nombres dans la chaîne.
    Séparateur par défaut : virgule ou saut de ligne.
    Séparateur custom : //SEP\n au début de la chaîne.
    Les nombres > 1000 sont ignorés.
    """
    if not numbers:
        return 0

    separator_pattern, body = _parse_separator(numbers)
    parts    = re.split(separator_pattern, body)
    integers = [int(p) for p in parts if p]

    _check_no_negatives(integers)

    return sum(n for n in integers if n <= 1000)


def _parse_separator(numbers: str) -> tuple[str, str]:
    if numbers.startswith("//"):
        header, _, body = numbers[2:].partition("\n")
        # Le séparateur peut être multi-caractères ou regex
        separator = re.escape(header)
        return separator, body
    return r"[,\n]", numbers


def _check_no_negatives(integers: list[int]) -> None:
    negatives = [n for n in integers if n < 0]
    if negatives:
        raise NegativeNumberException(negatives)
```

```python
# ── test_string_calculator.py ─────────────────────────────────────────────────

import pytest
from string_calculator import add, NegativeNumberException


class TestStringCalculator:
    # Etape 1 : chaîne vide
    def test_empty_string_returns_zero(self):
        assert add("") == 0

    # Etape 2 : un seul nombre
    def test_single_number_returns_its_value(self):
        assert add("5") == 5
        assert add("0") == 0

    # Etape 3 : deux nombres
    def test_two_numbers_returns_sum(self):
        assert add("1,2") == 3

    # Etape 4 : nombre quelconque de nombres
    def test_multiple_numbers(self):
        assert add("1,2,3,4") == 10

    # Etape 5 : saut de ligne comme séparateur
    def test_newline_as_separator(self):
        assert add("1\n2,3") == 6

    # Etape 6 : séparateur custom
    def test_custom_separator(self):
        assert add("//;\n1;2") == 3
        assert add("//|\n1|2|3") == 6
        assert add("//***\n1***2***3") == 6  # séparateur multi-caractères

    # Etape 7 : nombres négatifs
    def test_negative_numbers_raise_exception(self):
        with pytest.raises(NegativeNumberException) as exc_info:
            add("1,-2,-3")
        assert exc_info.value.negatives == [-2, -3]

    def test_single_negative_raises(self):
        with pytest.raises(NegativeNumberException):
            add("-5")

    # Etape 8 : nombres > 1000 ignorés
    def test_numbers_above_1000_are_ignored(self):
        assert add("2,1001") == 2
        assert add("1000,1001") == 1000  # 1000 inclus, 1001 exclus

    # Tests de robustesse
    def test_custom_separator_with_newlines_in_body(self):
        assert add("//;\n1;2\n3") == 6
```

---

#### Corrigé — Exercice B (choix des doubles)

1. **Repository avec données prédéfinies → Stub**  
   `InvoiceService.generate()` a besoin de données produit mais ne vérifie pas
   comment elles ont été obtenues. Un Stub retourne des données fixes sans
   logique. On n'utilise pas un Mock car on ne vérifie pas que `findById` a été
   appelé un certain nombre de fois — seul le résultat final importe.

2. **Vérifier qu'une méthode est appelée une fois → Mock**  
   L'assertion porte sur une *interaction* : `sendConfirmation` doit être appelée
   exactement une fois. C'est la définition d'un Mock. `MagicMock` de unittest
   avec `assert_called_once()`.

3. **Flux complet sans BDD → Fake**  
   Plusieurs méthodes du repository sont invoquées de manière cohérente (save,
   find, update). Un Fake (`InMemoryCartRepository`) maintient un état cohérent
   entre les appels — ce qu'un Stub ne peut pas faire.

4. **Vérifier les appels de log a posteriori → Spy**  
   On ne vérifie pas combien de fois `logger.warning()` a été appelé pendant
   le test, mais *après* son exécution. Un Spy enregistre les appels sans
   imposer d'assertion pendant l'exécution.

5. **Dix commandes de structures différentes → Fake ou PBT**  
   `InMemoryOrderRepository` avec dix ordres prédéfinis (Fake), ou mieux :
   property-based testing avec Hypothesis pour générer aléatoirement des
   commandes et vérifier que le CSV produit est toujours valide (colonnes
   présentes, valeurs bien échappées).

---
