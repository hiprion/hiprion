# AIR V3 — Average Intraday Range (Pine Script)

Indicateur TradingView reproduisant la logique de l'**AIR V3** de Romain Bailleul,
puis l'étendant : normalisation en % du prix, statistiques hors échantillon et
gestion propre des contrats futures continus.

Un seul fichier pour tous les instruments : **NQ, ES, MNQ, MES, contrats datés
(NQM2026) ou continus (ES1!, NQ1!), indices cash**.

Fichier : [`pine/air_v3.pine`](pine/air_v3.pine)

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

Ces mesures sont **normalisées en % du prix** par défaut (voir §4), puis résumées
par une tendance centrale (moyenne, médiane ou moyenne tronquée) et réappliquées
à l'open de la séance en cours :

```
AIR High = Open du jour x (1 + moyenne(High - Open) %)
AIR Low  = Open du jour x (1 - moyenne(Open - Low)  %)
```

Identité utile : `moy(High-Open) + moy(Open-Low) = moy(Range)`. La largeur de la
projection est donc toujours égale au range moyen — exactement ce que montre
l'AIR V3 d'origine :

```
200.25 + 163.25 = 363.50          (Avg High from Open + Avg Low from Open = Avg Range H-L)
30268.25 + 200.25 = 30468.50      (Proj. High)
30268.25 - 163.25 = 30105.00      (Proj. Low)
```

### Bloc 2 — Moyenne N séances de `(Open + High + Low) / 3`

Moyenne du **prix typique journalier** sur les N dernières séances, encadrée par
ses écarts-types (±1σ et ±2σ). Les zones colorées identifient les moments où le
marché est statistiquement **plus haut** (écart positif, zone verte) ou
**plus bas** (écart négatif, zone rouge) que sa moyenne habituelle.

---

## 2. Installation

1. TradingView → **Pine Editor**.
2. Coller le contenu de `pine/air_v3.pine`.
3. **Save**, puis **Add to chart**.
4. Se placer sur une unité de temps **intraday** (1 à 15 min recommandé).

---

## 3. Réglages par instrument

### ES1! / NQ1! (contrats continus) — configuration recommandée

| Réglage | Valeur |
|---|---|
| Séance | `Heures régulières (RTH)` |
| Fenêtre horaire | `0930-1600`, `America/New_York` |
| Back-adjustment des rolls | `Activé` |
| Unité de calcul | `% du prix` |

Ce sont les valeurs par défaut : l'indicateur est prêt à l'emploi sur ES1!.

### Contrats datés (NQM2026, ESZ2026…)

Identique, le back-adjustment étant simplement sans effet. Passer la séance sur
`Séance complète (ETH)` si l'on veut retrouver le comportement d'un AIR calculé
sur la bougie journalière complète.

---

## 4. Ce que ce script fait différemment de l'AIR V3 d'origine

### Session RTH sans contrainte d'historique

La bougie journalière d'ES1! et NQ1! **commence à 18:00 ET la veille** : l'open de
référence est un open Globex de nuit et l'AIR mesure une excursion sur ~23 h, ce
qui élargit beaucoup les bandes pour du day trading.

Le mode `Heures régulières (RTH)` lit des bougies journalières **RTH** via
`ticker.modify(..., session = session.regular)`. Contrairement à une
reconstruction depuis les bougies du graphique (qui exigerait ~23 000 bougies M1
pour 60 séances), il n'y a **aucune contrainte d'historique**.

### Contrats continus : gaps de roll neutralisés

`ES1!` enchaîne les contrats sans ajustement : chaque roll trimestriel crée un
saut de prix (base de portage). Les ranges intraday n'en souffrent pas, mais la
**moyenne N séances est décalée et son écart-type artificiellement gonflé**.
`backadjustment.on` réaligne l'historique sur le contrat courant. Sans effet sur
les symboles non-futures.

`settlement_as_close` permet par ailleurs d'utiliser le settlement officiel comme
clôture journalière (utile pour le mode « Clôture veille ± Range/2 »).

### Normalisation en % du prix

Sur 60 séances le niveau de l'indice bouge de plusieurs pourcents : moyenner des
ranges en points mélange des régimes de prix différents et **sous-estime le range
courant en marché haussier**. Le calcul se fait donc en `(H−O)/O` puis se
réapplique à l'open du jour.

Effet secondaire utile : les réglages deviennent transposables d'un instrument à
l'autre sans retouche (ES ≈ 65-90 pts de range, NQ ≈ 360 pts, mais ~1 % des deux
côtés). Le mode `Points` reste disponible pour reproduire le calcul brut.

### Taux de touche hors échantillon

L'AIR V3 d'origine compare chaque séance à une moyenne **qui la contient** — un
taux in-sample, légèrement optimiste. Ici, chaque séance est comparée à la moyenne
des N séances qui la **précèdent** (walk-forward) : c'est une vraie mesure de
fiabilité prospective. Le tableau affiche `hors échantillon (in-sample)`.

Sur données simulées asymétriques l'écart est net : 47 % in-sample contre 40 %
hors échantillon. La ligne nécessite 2 × N séances d'historique ; en dessous elle
affiche `—` et seule la valeur in-sample reste disponible.

---

## 5. Paramètres

### 1 · Calcul
| Paramètre | Défaut | Rôle |
|---|---|---|
| Séances analysées | 60 | profondeur de l'échantillon |
| Unité de calcul | % du prix | ou `Points` (calcul brut d'origine) |
| Tendance centrale | Moyenne | Moyenne / Médiane / Moyenne tronquée |
| Troncature (%) | 10 | % écarté de chaque côté en mode tronqué |

> **Médiane / moyenne tronquée** : recommandées sur indices, dont la distribution
> des ranges est fortement asymétrique (quelques séances extrêmes tirent la
> moyenne vers le haut).

### 2 · Séance analysée
`Heures régulières (RTH)` · `Séance complète (ETH)` · `Fenêtre personnalisée`,
plus la fenêtre horaire et son fuseau.

En mode RTH la fenêtre horaire ne sert qu'au **suivi de la séance en cours** sur
le graphique ; les statistiques viennent des bougies journalières RTH. Les deux
doivent coïncider : vérifier visuellement que l'open tracé tombe bien sur la
bougie d'ouverture.

`Fenêtre personnalisée` (pour une plage arbitraire, ex. `0930-1130`) reconstruit
l'historique depuis les bougies du graphique : il faut alors suffisamment
d'historique chargé, et un graphique intraday.

### 3 · Futures
Back-adjustment des rolls, settlement comme clôture.

### 4 · Bandes AIR
Mode de bandes, bandes d'extension (percentile ou +1σ), zone d'équilibre autour
de l'open, niveaux Fibonacci.

### 5 · Moyenne N séances
Longueur, affichage des écarts, multiplicateurs σ.

### 6 · Affichage
Tableau, position/taille, étiquettes, prolongement à droite, couleurs.

---

## 6. Lecture du tableau

| Ligne | Interprétation |
|---|---|
| **Séance analysée** | rappel du mode et de l'unité de calcul |
| **Séances** | taille d'échantillon réelle ; `45 / 60 ⚠` signale un historique insuffisant |
| **Range moyen H-L** | amplitude moyenne, en points **et** en % du prix |
| **Moy. High - Open / Open - Low** | excursions moyennes de part et d'autre de l'open |
| **% touche haute / basse** | fréquence d'atteinte hors échantillon, in-sample entre parenthèses (≈ 40 % : normal, la distribution est asymétrique) |
| **Range utilisé** | part du range moyen déjà consommée — au-delà de 100 %, séance statistiquement étendue |
| **Proj. H / L** | les deux bandes AIR du jour |
| **Extension H / B** | seuils d'excursion rarement dépassés |
| **Excursion haute / basse utilisée** | % de l'excursion moyenne consommée, avec le **z-score** (> +2 = extrême) |
| **Écart vs moyenne** | distance du prix à la moyenne N séances, en points et en % |
| **Z-score / % séances >** | écart normalisé, et part des N séances au-dessus de la moyenne |

### Utilisation typique
- Range utilisé faible + prix proche de l'open → potentiel d'expansion restant.
- Prix sur l'AIR High avec ~100 % du range moyen consommé → poursuite haussière
  statistiquement moins probable (prise de profit / contre-tendance).
- Prix au-delà de +2σ de la moyenne N séances → survalorisation statistique.

---

## 7. Alertes

- AIR High atteint / AIR Low atteint
- Extension atteinte
- Écart positif extrême / Écart négatif extrême (±2σ vs moyenne N séances)

---

## 8. Notes techniques

- **Aucun repaint** : toutes les statistiques sont calculées exclusivement sur des
  séances **clôturées** (décalages `[1]` et suivants). `request.security(...,
  lookahead_on)` combiné à ces décalages renvoie une valeur connue dès
  l'ouverture et figée pour toute la séance.
- L'open, le high et le low de la séance en cours sont suivis **nativement** sur
  l'unité de temps du graphique : identiques en historique et en temps réel.
- Le taux hors échantillon utilise une somme glissante (coût linéaire) et lit
  2 × N séances journalières.
- Unités de temps supérieures au journalier : non pertinentes, l'indicateur le
  signale.

---

## 9. Limites

Ce sont des **statistiques descriptives**, pas des signaux : les bandes indiquent
où la séance se situe par rapport à son comportement habituel, elles ne prédisent
ni un retournement ni une continuation. Les jours à catalyseur (CPI, FOMC, NFP,
quadruple witching) sortent régulièrement des bandes — la médiane ou la moyenne
tronquée les rend plus robustes.

Pistes non implémentées, par ordre d'intérêt : profil de complétion du range par
tranche horaire (projection qui se resserre au fil de la séance + probabilité de
touche conditionnelle au temps restant), conditionnement au gap d'ouverture,
pondération par régime de volatilité (ATR10/ATR60 ou VIX), exclusion des
demi-séances et des jours fériés.
