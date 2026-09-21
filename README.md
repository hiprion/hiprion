# AIR V3 — Average Intraday Range (Pine Script)

Indicateur TradingView reproduisant la logique de l'**AIR V3** de Romain Bailleul,
pensé pour le Nasdaq (NQ / NQM2026, MNQ, US100…) mais utilisable sur n'importe
quel instrument.

Fichier : [`pine/air_v3_nasdaq.pine`](pine/air_v3_nasdaq.pine)

---

## 1. Principe de calcul

### Bloc 1 — Bandes AIR (projection de la séance)

Sur les **N dernières séances clôturées** (60 par défaut), l'indicateur mesure
trois distributions à partir des données journalières **Open / High / Low** :

| Mesure | Formule | Signification |
|---|---|---|
| `Range` | `High - Low` | amplitude totale de la séance |
| `High - Open` | `High - Open` | excursion **haussière** depuis l'open |
| `Open - Low` | `Open - Low` | excursion **baissière** depuis l'open |

On en extrait une tendance centrale (moyenne, médiane ou moyenne tronquée), que
l'on projette sur l'open de la séance en cours :

```
AIR High = Open du jour + moyenne(High - Open)
AIR Low  = Open du jour - moyenne(Open - Low)
```

Identité utile : `moy(High-Open) + moy(Open-Low) = moy(Range)`. La largeur de la
projection est donc toujours égale au range moyen — c'est exactement ce que montre
l'AIR V3 d'origine :

```
200.25 + 163.25 = 363.50          (Avg High from Open + Avg Low from Open = Avg Range H-L)
30268.25 + 200.25 = 30468.50      (Proj. High)
30268.25 - 163.25 = 30105.00      (Proj. Low)
```

### Bloc 2 — Moyenne N séances de `(Open + High + Low) / 3`

Moyenne du **prix typique journalier** sur les N dernières séances, encadrée par
ses écarts-types (±1σ et ±2σ). Les zones colorées identifient les moments où le
Nasdaq est statistiquement **plus haut** (écart positif, zone verte) ou
**plus bas** (écart négatif, zone rouge) que sa moyenne habituelle.

---

## 2. Installation

1. TradingView → **Pine Editor** (en bas de l'écran).
2. Coller le contenu de `pine/air_v3_nasdaq.pine`.
3. **Save**, puis **Add to chart**.
4. Se placer sur une unité de temps **intraday** (1 à 15 min recommandé) — les
   projections n'ont de sens qu'à l'intérieur d'une séance.

---

## 3. Paramètres

### 1 · Calcul
| Paramètre | Défaut | Rôle |
|---|---|---|
| Séances analysées | 60 | profondeur de l'échantillon statistique |
| Tendance centrale | Moyenne | Moyenne / Médiane / Moyenne tronquée |
| Troncature (%) | 10 | % écarté de chaque côté en mode tronqué (neutralise CPI, FOMC, NFP…) |
| Définition de la séance | Bougie journalière (D) | ou plage horaire personnalisée |
| Session personnalisée | 0930-1600 | pour ne travailler que le RTH, par exemple |

> **Médiane / moyenne tronquée** : recommandées sur le Nasdaq, dont la distribution
> des ranges est fortement asymétrique (quelques séances extrêmes tirent la moyenne
> vers le haut).

### 2 · Bandes AIR
| Paramètre | Rôle |
|---|---|
| Mode bandes | `Open-based` (mode AIR V3 d'origine), `Symétrique (Range/2)`, `Clôture veille ± Range/2` |
| Bandes d'extension | zone rarement atteinte : percentile 80 ou moyenne + 1σ |
| Zone d'équilibre autour de l'open | bande verte `Open ± x %` du range moyen |
| Niveaux Fibonacci | 0.236 / 0.382 / 0.5 / 0.618 / 0.786 de la projection |

### 3 · Moyenne N séances
Longueur, affichage des écarts, multiplicateurs σ.

### 4 · Affichage
Tableau, position/taille, étiquettes de prix, prolongement à droite, couleurs.

---

## 4. Lecture du tableau

| Ligne | Interprétation |
|---|---|
| **Range moyen H-L** | amplitude moyenne d'une séance |
| **Moy. High - Open / Open - Low** | excursions moyennes au-dessus / en dessous de l'open |
| **% touche bande haute / basse** | fréquence historique d'atteinte de chaque bande (≈ 40-45 % : c'est normal, la distribution est asymétrique) |
| **Range utilisé** | part du range moyen déjà consommée aujourd'hui — au-delà de 100 %, la séance est statistiquement étendue |
| **Proj. H / L** | les deux bandes AIR du jour |
| **Extension H / B** | seuils d'excursion rarement dépassés |
| **Excursion haute / basse utilisée** | % de l'excursion moyenne consommée, avec le **z-score** entre parenthèses (> +2 = extrême) |
| **Écart vs moyenne** | distance du prix à la moyenne N séances, en points et en % |
| **Z-score / % séances >** | écart normalisé, et part des N séances au-dessus de la moyenne |

### Utilisation typique
- Range utilisé faible + prix proche de l'open → potentiel d'expansion restant.
- Prix sur l'AIR High avec ~100 % du range moyen consommé → poursuite haussière
  statistiquement moins probable (zone de prise de profit / contre-tendance).
- Prix au-delà de +2σ de la moyenne 60 séances → survalorisation statistique.

---

## 5. Alertes disponibles

- AIR High atteint / AIR Low atteint
- Extension atteinte
- Écart positif extrême / Écart négatif extrême (±2σ vs moyenne N séances)

---

## 6. Notes techniques

- **Aucun repaint** : toutes les statistiques sont calculées exclusivement sur des
  séances **clôturées** (décalages `[1]` et suivants). En mode journalier,
  `request.security(..., lookahead_on)` combiné à ces décalages renvoie une valeur
  connue dès l'ouverture et figée pour toute la séance.
- L'open, le high et le low de la séance en cours sont suivis **nativement** sur
  l'unité de temps du graphique : identiques en historique et en temps réel.
- Le mode **session personnalisée** construit son historique à partir des bougies
  du graphique : il faut donc suffisamment d'historique chargé (60 séances de RTH
  en 1 min ≈ 23 000 bougies, ce qui peut dépasser la limite d'un compte gratuit).
  Le mode **bougie journalière** n'a pas cette contrainte.
- Unités de temps supérieures au journalier : non pertinentes, l'indicateur le signale.

---

## 7. Limites

Ce sont des **statistiques descriptives**, pas des signaux : les bandes indiquent
où la séance se situe par rapport à son comportement habituel, elles ne prédisent
ni un retournement ni une continuation. Les jours à catalyseur (CPI, FOMC, NFP,
quadruple witching) sortent régulièrement des bandes — la médiane ou la moyenne
tronquée les rend plus robustes.
