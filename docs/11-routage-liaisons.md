# 11 — Routage du PCB

Comment les liaisons sont faites, et pourquoi. Tout le routage est produit par
le générateur du projet : relancer le
script redonne le même résultat.

**Hypothèses** : 24 V, 10 A RMS par moteur, cuivre 70 µm dehors, isolement
6 mil (0,15 mm).

## 1. Rôle des couches

```
 L1  Top      composants, zones de puissance, signaux
 L2  GND      plan de masse
 L3  VBUS     plan d'alimentation 24 V
 L4  VBUS     plan d'alimentation, avec une zone 3,3 V sous le MCU
 L5  GND      plan de masse
 L6  Bottom   zones de phase sous les ponts, signaux
```

- **Plans pleins pour GND et VBUS** : aucun courant de puissance en piste.
  L'aller (VBUS) et le retour (GND) sont superposés à quelques dixièmes de
  millimètre, ce qui donne une boucle de commutation de l'ordre du nanohenry et
  peu de surtensions.
- **Chaque couche de signal a un plan de masse juste à côté** (L1/L2, L6/L5) :
  le courant de retour passe sous la piste.
- **Zone 3,3 V sur L4** : les 35 pastilles du 3,3 V numérique, autour du MCU,
  se raccordent par un via au lieu d'un long arbre de pistes. L3 reste un plan
  VBUS complet.

## 2. Raccordement aux plans

| Cas | Traitement | Pourquoi |
|---|---|---|
| Grande pastille (languette de MOSFET, shunt, pad thermique) | grille de vias dans la pastille, au pas de 1 mm | ≈ 1,2 A par via (IPC-2221) : 16 vias en parallèle portent le courant et évacuent la chaleur |
| Petite pastille (découplage, CI) | via juste à côté + piste courte | pas de via dans une petite pastille (la soudure fuirait) ; chaque millimètre de piste ajoute ≈ 1 nH |
| Pastille traversante | rien à ajouter | elle traverse déjà toutes les couches |

Vias de 0,6 / 0,3 mm, **recouverts de vernis**. Les vias dans les pastilles
doivent être bouchés à la fabrication (voir la note de fabrication).

## 3. Puissance des ponts

Le connecteur de phase est au bord de la carte, derrière les MOSFET hauts. En
surface, le nœud de phase ne peut pas le rejoindre sans croiser VBUS. Il passe
donc par la face inférieure.

| Segment | Couche | Réalisation |
|---|---|---|
| MOSFET haut ↔ MOSFET bas (nœud de phase) | L1 | zone de cuivre entre les deux rangées |
| Nœud de phase → connecteur | L6 | zone de ≈ 9 mm de large sous les MOSFET hauts |
| Passage L1 → L6 | vias | **36 vias par phase** |
| MOSFET bas → shunt | L1 | zone de cuivre directe ; pastilles de mesure du shunt séparées |

Dimensionnement à 10 A RMS :
- **zone L6** : ≈ 0,75 mΩ, soit ≈ 75 mW de pertes ; elle tolère ≈ 26 A pour 20 °C d'échauffement (IPC-2221) ;
- **36 vias** : ≈ 43 A au total, et une inductance inférieure à 0,1 nH.

Les zones sont décrites une seule fois et reproduites pour les 4 moteurs. Les
pastilles de puissance y sont reliées **en plein** : des rayons thermiques
(4 × 10 mil) ne porteraient pas le courant.

## 4. Signaux

| Liaison | Largeur | Règle |
|---|---|---|
| Grilles (DRV8305 → résistance → MOSFET) | 0,3 mm | la plus courte possible ; le retour (source) suit l'aller |
| Mesures SH / SL vers le DRV8305 | 0,2 mm | prises au pied du MOSFET, pas sur la zone de puissance |
| Paires Kelvin des shunts | 0,2 mm | côte à côte jusqu'au DRV8305, sans via, loin des nœuds de phase |
| Alimentations 3,3 V / 5 V | 0,3 mm | — |
| Logique (SPI, I2C, UART, CAN) | 0,2 mm | au-dessus d'un plan de masse continu ; CAN en paire différentielle |

**Routeur.** Les signaux sont routés par un routeur sur grille écrit pour le
projet :
- **recherche A\*** sur une grille de 0,05 mm, sur L1 et L6 ;
- obstacles gonflés de l'isolement : le DRC est respecté par construction ;
- **liaisons courtes et critiques d'abord** : grilles, mesures, Kelvin, puis la logique ;
- **éclatement des broches** des boîtiers au pas de 0,5 mm, pour que chaque broche garde une sortie ;
- **arrachage-reroutage** : un net bloqué peut déplacer un net déjà posé, qui est rerouté ensuite (principe de *PathFinder*).

Le routage des signaux de logique est **en cours** ([10 §7](10-conception-pcb.md)).

## 5. Vérification

Après chaque génération : recoulage des plans dans Altium, puis DRC complet
(isolement, court-circuit, largeur, vernis, sérigraphie, antennes).

## 6. Points à valider avant fabrication

- vias dans les pastilles bouchés et recouverts (IPC-4761 type VII) ;
- échauffement des zones L6 si le courant passe à 40 A par moteur ;
- soudure des connecteurs de puissance reliés en plein : fer puissant ou préchauffage.
