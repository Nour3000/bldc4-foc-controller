# 10 — Conception du PCB

Suite de 09 (schéma). Les choix de routage
sont détaillés et chiffrés dans [11](11-routage-liaisons.md).

**État** : placement, plans, puissance des ponts, grilles et mesures routés et
validés par le DRC. Routage des signaux de logique en cours.

## 1. Méthode : PCB généré au format ASCII d'Altium

Le PCB est produit par un script Python, comme le
schéma. Altium sait enregistrer et relire un PCB au **format ASCII**
(enregistrements `Board`, `Component`, `Pad`, `Track`, `Via`, `Net`, `Class`,
`Polygon`, `DXPRule`).

L'empilage des couches est décrit par des dizaines de champs liés entre eux.
Plutôt que de l'écrire à la main, il vient d'un **modèle créé dans Altium**
(empilage 6 couches réglé dans le *Layer Stack Manager*, puis enregistré en
ASCII).
Le générateur reprend ces enregistrements, ajuste les noms et épaisseurs des
couches, puis ajoute composants, pastilles, nets, plans et règles. Chaque
composant porte l'identifiant unique de son symbole : schéma et PCB restent
synchronisés.

## 2. Empreintes

Règle suivie : **aucune cote de mémoire**.
- **Circuits à pad thermique et modules** : motif recommandé par le fabricant (*Example Board Layout*).
- **Boîtiers normalisés** (LQFP, passifs) : calcul IPC-7351 à partir des cotes JEDEC.
- **Connecteurs** sans plan exploitable : cotes standard, **à valider** (§7).

| Empreinte | Composant | Motif retenu (mm) | Source |
|---|---|---|---|
| `HTQFP-48_PAD` | DRV8305 | 48 × 1,6 × 0,3, pas 0,5, centres à ±4,25 ; pad 49 : 5,17 × 5,17 + vias Ø 0,2 au pas de 1,1 | TI PHP0048N |
| `SON-8_5x6` | CSD18540Q5B | drain 4,44 × 4,52 avec 4 doigts ; broches 1-4 : 1,1 × 0,71 au pas de 1,27 | TI SLPS488B §7.2 |
| `HSOIC-8_PAD` | LMR33630 | 8 × 1,55 × 0,6, pas 1,27 ; pad 9 : 2,6 × 3,1 + vias | TI DDA0008J |
| `WSON-6` | TMP117 | 6 × 0,45 × 0,3, pas 0,65, sans pad thermique | TI DRV0006B |
| `SOT-23-5` / `SOT-23` / `SOIC-8` / `SOD-523` | LDO, EEPROM, ESD, TCAN1042 | motifs TI DBV / DBZ / D / DYA | TI |
| `ESP32-C3-MINI-1` | module | 48 pastilles 0,4 × 0,8 au pas de 0,8, 4 coins, 9 pads de masse ; antenne en débord | Espressif v2.2 |
| `R2512_4T` | shunt WSK2512 | courant : 2,29 × 3,30 ; mesure : 1,70 × 0,76 | Vishay doc. 30108 |
| `L_7x7` | XAL7070-682 | 2 × 1,63 × 5,93 à 4,57 d'entraxe | Coilcraft doc. 856-4 |
| `LQFP-100_14x14` | STM32G474VET6 | 100 × 1,65 × 0,30, pas 0,5, centres à ±7,625 | IPC-7351 |
| `CP_Radial_D10_P5.0` | bus DC | Ø 10, pas 5,0, perçage Ø 1,0 | EIA radial |

Deux cotes demandaient une lecture attentive :
- **DRV8305** : la cote 8,5 de TI est un **entraxe** de pastilles, pas une emprise extérieure. Lue autrement, les pastilles d'angle se recouvraient.
- **ESP32** : le motif compte 61 pastilles pour 53 broches. Il y a 8 pads de masse supplémentaires sous le module, rattachés au pad central.

## 3. Corrections de composants

Trouvées en relisant les datasheets pour les empreintes :

| Problème | Correction |
|---|---|
| Shunt annoncé 3 W ; la datasheet Vishay donne **1 W** | « 5 mΩ 1 % 1 W » : 0,5 W à 10 A RMS, mais l'hypothèse 40 A par moteur est exclue avec ce composant |
| Référence de shunt mal formée | `WSK25125L000FEA` (codification Vishay de 5 mΩ) |
| Inductance XAL7070-822 inexistante (la gamme s'arrête à 6,8 µH) | XAL7070-682ME ; ondulation du buck 1,46 A crête-crête, acceptable pour 3 A |
| Condensateur polymère CMS sans motif disponible | radial traversant Ø 10 mm, 470 µF 35 V, faible ESR, ≥ 4 A d'ondulation |

## 4. Placement

| Élément | Choix |
|---|---|
| Carte | **190 × 120 mm**, 6 couches, 4 trous M3 reliés à GND |
| Empilage | L1 Top · L2 GND · L3 VBUS · L4 VBUS / 3V3_D · L5 GND · L6 Bottom ; 70 µm dehors, 35 µm dedans ; ≈ 1,6 mm (stackup) |
| Zones | colonne gauche : entrée 24 V et banc bus DC · 4 cellules onduleur de 70 × 48 mm en haut et en bas · bande centrale : MCU et alimentations logiques · colonne droite : CAN, I2C, ESP32 |

**Cellule onduleur**, conçue une fois, posée telle quelle en haut et tournée de
180° en bas : les 4 moteurs ont une implantation identique.

| Élément | Position locale (x ; profondeur depuis le bord) | Raison |
|---|---|---|
| Connecteur de phases | (25 ; 12), au ras du bord | sortie des câbles moteur |
| MOSFET hauts / bas | x = 12, 25, 38 ; profondeur 24 / 35 | nœud de phase de 4 mm entre les deux |
| Shunts A et B | sous les MOSFET bas | boucle MOSFET bas → shunt → GND la plus courte |
| DRV8305 | (56 ; 28), sorties de grille vers les ponts | liaisons de grille courtes |
| Résistances de grille | à 3,9 mm de chaque MOSFET | au plus près de la grille |
| NTC | contre le MOSFET de la phase C | mesure de l'étage le plus chaud |

Le générateur vérifie qu'aucune zone de placement ne se superpose, trous de
fixation compris : 283 composants, 1 159 pastilles, 245 nets.

## 5. Règles de conception

| Règle | Valeur | Justification |
|---|---|---|
| Isolement | **6 mil (0,15 mm)** | compatible avec le pas de 0,5 mm (7,9 mil entre pastilles du STM32) ; au-dessus du minimum IPC-2221 pour 30 V |
| Largeur de piste | 6 à 400 mil, préférée 10 mil | 6 mil = classe de fabrication visée ; 400 mil pour les pistes de puissance |
| Vernis | expansion 1,5 mil, pont minimal 4 mil | le pont par défaut de 10 mil est impossible au pas de 0,5 mm |
| Vias | Ø 0,6 / perçage 0,3 mm, **recouverts de vernis** | sans recouvrement, les ouvertures de vias voisins laissent des filets de vernis trop fins |
| Perçage maximal | 4 mm | XT60 Ø 2,7 mm et trous M3 Ø 3,2 mm |
| Connexion aux zones de cuivre | **en plein** pour les vias et la classe POWER ; rayons thermiques pour le reste | des rayons de 4 × 10 mil ne portent pas un courant de phase ; à 1 mm de pas, les rayons de vias voisins se recouvrent et isolent des vias |
| Couches de routage | L1 et L6 seulement | L2 à L5 sont des plans |
| Sérigraphie | à 0,4 mm de toute pastille | le vernis est déjà ouvert de 0,1 mm autour des pastilles |

16 désignateurs (5,6 % des composants) n'ont pas de place en sérigraphie dans
les zones les plus denses. Ils sont sur le calque d'assemblage, comme sur les
cartes denses du commerce.

## 6. Vérifications DRC : problèmes trouvés et corrections

| Constat | Cause | Correction |
|---|---|---|
| Sérigraphie à 3,65 mil du vernis | traits d'empreinte à 0,15 mm du cuivre, vernis ouvert de 0,1 mm | découpe automatique de la sérigraphie à 0,4 mm de toute pastille |
| Ponts de vernis trop fins sur le STM32 | règle de 10 mil incompatible avec le pas de 0,5 mm | expansion 1,5 mil, pont 4 mil |
| Court-circuit entre un trou M3 et un connecteur d'encodeur | le contrôle de placement ignorait les trous de fixation | connecteur décalé de 5 mm ; trous ajoutés au contrôle |
| Désignateurs superposés | taille des textes sous-estimée | placement automatique des désignateurs (10 positions, 3 tailles), écart 0,38 mm |
| Vias de drain isolés du plan VBUS (« antennes ») | rayons thermiques à 1 mm de pas : le cuivre restant est trop étroit et supprimé | connexion en plein pour les vias |
| Erreur de requête de règle au recoulage | `IsSMTPad` n'existe pas dans le langage de requêtes d'Altium | portée réduite à `IsVia Or InNetClass('POWER')`, qui suffit (§5) |
| Pistes s'arrêtant à côté de pastilles rondes | zone d'arrivée prise comme le rectangle englobant de la pastille | arrivée limitée au disque de cuivre |

Résultat après corrections : **0 violation** sur toutes les règles
(isolement, court-circuit, largeur, vernis, sérigraphie, perçage, antennes),
hors connexions de logique restant à router.

## 7. Routage

Détails et calculs : [11-routage-liaisons.md](11-routage-liaisons.md).

| Étape | Contenu | Résultat |
|---|---|---|
| 1. Plans | pastilles GND / VBUS / 3V3_D vers les plans internes | 204 vias dans les grandes pastilles, 296 vias de sortie |
| 2. Puissance | nœuds de phase (L1 → 36 vias → L6 → connecteur), sources basses vers shunts | 36 zones de cuivre, 448 vias de transfert, DRC sans erreur |
| 3. Signaux | grilles, mesures SH/SL, paires Kelvin, logique | routeur du générateur avec arrachage-reroutage ; **en cours** |

Évolution du routage des signaux :

| Version | Modification | Liaisons routées | Nets incomplets |
|---|---|---|---|
| 1 | grille 0,1 mm | 175 | 116 |
| 2 | grille 0,05 mm, deux passes | 214 | 104 |
| 3 | grilles avant Kelvin | 225 | 81 |
| 4 | liaisons courtes d'abord, plan 3V3_D partagé sur L4 | 231 | 77 |
| 5 | arrachage-reroutage, éclatement des broches au pas fin | en cours | — |

L'autorouteur d'Altium a été essayé sur les nets restants, puis écarté. Il ne
reconnaît pas les plans réalisés en polygones sur des couches de signal, et
son résultat ajoutait des violations d'isolement et découpait les plans.

## 8. Points à valider avant fabrication

| Point | Pourquoi |
|---|---|
| Empreintes XT60PW-M, MR30PW-M, PicoBlade 53048, TC2030 | plans fabricant peu lisibles ou indisponibles : à vérifier sur pièce |
| Pads centraux de l'ESP32, motif du WSK2512 | interprétations du §2 à recouper avec les fichiers du fabricant |
| Vias dans les pastilles des MOSFET et des shunts | à commander bouchés et recouverts (IPC-4761 type VII) |
| Échauffement des zones de phase sur L6 | à recalculer si le courant passe à 40 A par moteur |
