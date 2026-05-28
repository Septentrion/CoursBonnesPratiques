Voici un exemple concret en **PHP** (qui supporte nativement la covariance et la contravariance depuis PHP 7.4) avec des classes `Animal` (mère) et `Chien` (fille), illustrant les deux concepts dans la même hiérarchie.

## Définitions rapides

| Type | Ce qui change dans la classe fille |
| :-- | :-- |
| **Covariance** | Le **type de retour** est plus **spécifique** (sous-type) [^4][^6][^9] |
| **Contravariance** | Le **type de paramètre** est plus **général** (super-type) [^4][^6][^9] |


***

## Exemple complet

```php
<?php

// Hiérarchie de types pour les paramètres (contravariance)
class Nourriture {
    public functionToString(): string | Stringable {
        return "Nourriture générique";
    }
}

class NourritureAnimal extends Nourriture {
    public function toString(): string {
        return "Nourriture pour animal";
    }
}

class NourritureChien extends NourritureAnimal {
    public function toString(): string {
        return "Croquettes pour chien";
    }
}

// Hiérarchie de types pour les retours (covariance)
class Animal {
    public string $nom;
    
    public function __construct(string $nom) {
        $this->nom = $nom;
    }
    
    // Méthode de base : retourne Animal
    public function faireBruit(): Animal {
        echo "{$this->nom} fait un bruit d'animal\n";
        return $this;
    }
    
    // Méthode de base : accepte NourritureAnimal (type spécifique)
    public function manger(NourritureAnimal $nourriture): void {
        echo "{$this->nom} mange " . $nourriture->toString() . "\n";
    }
}

class Chien extends Animal {
    
    // COVARIANCE : type de retour PLUS SPÉCIFIQUE (Chien au lieu d'Animal)
    public function faireBruit(): Chien {
        echo "{$this->nom} aboie : Wouf !\n";
        return $this;
    }
    
    // CONTRVARIANCE : type de paramètre PLUS GÉNÉRAL (Nourriture au lieu de NourritureAnimal)
    public function manger(Nourriture $nourriture): void {
        echo "{$this->nom} (chien) mange " . $nourriture->toString() . "\n";
    }
}

// === Utilisation ===

$animal = new Animal("Médor");
$chien = new Chien("Rex");

// Covariance en action : on peut affecter Chien à Animal
$animal2 = $chien->faireBruit();  // Retourne Chien, mais affecté à Animal ✓

// Contravariance en action : Chien accepte plus de types
$animal->manger(new NourritureAnimal());  // ✓ NourritureAnimal
$chien->manger(new NourritureAnimal());   // ✓ NourritureAnimal
$chien->manger(new Nourriture());         // ✓ Nourriture (plus général !)
// $animal->manger(new Nourriture());     // ✗ ERREUR : Nourriture pas accepté par Animal
```


***

## Tableau récapitulatif des différences

| Aspect | Classe mère `Animal` | Classe fille `Chien` | Concept |
| :-- | :-- | :-- | :-- |
| **`faireBruit()` retour** | `: Animal` | `: Chien` | **Covariance** [^4][^6] |
| **`manger()` paramètre** | `NourritureAnimal $nourriture` | `Nourriture $nourriture` | **Contravariance** [^4][^6] |
| **Spécificité du retour** | Type général | Type plus spécifique | Covariance = spécialisation |
| **Spécificité du paramètre** | Type spécifique | Type plus général | Contravariance = généralisation |


***

## Règle mnémotechnique

- **Covariance** → **CO**mme la hiérarchie (retour suit la même direction : `Animal` → `Chien`)[^2][^3]
- **Contravariance** → **CON**tre la hiérarchie (paramètre va dans la direction opposée : `NourritureAnimal` ← `Nourriture`)[^3][^2]

En résumé : une méthode fille peut **retourner un type plus précis** (covariance) et **accepter un type plus large** (contravariance) que la méthode mère, tant que la sécurité des types est préservée.[^4][^9]
<span style="display:none">[^1][^10][^5][^7][^8]</span>

<div align="center">⁂</div>

[^1]: https://learn.microsoft.com/fr-fr/dotnet/csharp/programming-guide/concepts/covariance-contravariance/

[^2]: http://blog.mahamadoutoure.com/2015/03/covariance-contravariance-et-invariant.html

[^3]: https://joow.github.io/2018/08/26/invariance-covariance-contravariance.html

[^4]: https://runebook.dev/fr/docs/php/language.oop5.variance

[^5]: https://www.reddit.com/r/rust/comments/6p6w1d/what_are_covariance_and_contravariance_xpost_from/

[^6]: https://phpsources.net/Manuel-PHP-Janv-2019/language.oop5.variance.html

[^7]: https://learn.microsoft.com/fr-fr/dotnet/standard/generics/covariance-and-contravariance

[^8]: https://openclassrooms.com/forum/sujet/covariance-des-variable-et-polymorphisme

[^9]: https://www.php.net/manual/fr/language.oop5.variance.php

[^10]: https://vc.19633.com/c3-2/1002017160.html
