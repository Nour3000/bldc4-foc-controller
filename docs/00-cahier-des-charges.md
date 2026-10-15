# Cahier des charges

Exigences numérotées, référencées par les documents de conception et par le
plan de validation (08). Statut : ✅ figé,
⚠️ hypothèse par défaut à confirmer.

## Fonctionnelles

| ID | Exigence | Statut |
|---|---|---|
| REQ-F01 | Piloter **4 moteurs BLDC/PMSM** indépendants en commande vectorielle (FOC) | ✅ |
| REQ-F02 | Boucle de courant à **20 kHz**, boucle de vitesse à **1 kHz** | ✅ |
| REQ-F03 | Position rotor absolue par encodeur magnétique 14 bits, par moteur | ✅ |
| REQ-F04 | Recevoir les consignes couple / vitesse par **CAN FD** — seul chemin de commande | ✅ |
| REQ-F05 | Remonter la télémétrie vers le cloud (WiFi/MQTT) **sans** pouvoir commander un mouvement | ✅ |
| REQ-F06 | Mesure de température : ambiance carte + chaque étage de puissance | ✅ |
| REQ-F07 | Stocker la calibration (offsets CSA, zéro rotor) et une identité unique non volatile | ✅ |
| REQ-F08 | Utiliser les quatre familles de bus : SPI, I2C, UART, CAN | ✅ |

## Performances

| ID | Exigence | Statut |
|---|---|---|
| REQ-P01 | Tension de bus **24 V** nominal (18–30 V de fonctionnement) | ✅ |
| REQ-P02 | **40 A agrégé, 10 A RMS par moteur** | ⚠️ ou 40 A par moteur, cf. 01 C-3 |
| REQ-P03 | Bande passante de courant ≥ 1 kHz, marge de phase ≥ 45° | ✅ ([05](05-reglage-boucles.md) : 63°) |
| REQ-P04 | Charge CPU de la boucle de courant ≤ 30 % (4 axes) | ✅ (estimation ~17 %, 02 §5) |
| REQ-P05 | Latence consigne CAN → nouvelle référence de courant ≤ 1 ms | ✅ |

## Sûreté de fonctionnement

| ID | Exigence | Statut |
|---|---|---|
| REQ-S01 | Défaut driver → coupure PWM **matérielle**, sans logiciel (entrée BREAK) | ✅ |
| REQ-S02 | Surintensité logicielle → défaut en < 1 période (50 µs) | ✅ |
| REQ-S03 | Perte de consigne CAN pendant > 100 ms → arrêt de l'axe | ✅ |
| REQ-S04 | Trame encodeur corrompue → rejet (parité), extrapolation, défaut après N | ✅ |
| REQ-S05 | Un redémarrage de l'ESP32 ne perturbe pas le contrôle moteur | ✅ |
| REQ-S06 | Toute consigne non finie (NaN, ±∞) est rejetée avant le contrôle | ✅ |
| REQ-S07 | Étages de puissance inhibés au reset (EN_GATE tiré bas) | ✅ |

## Contraintes

| ID | Contrainte |
|---|---|
| REQ-C01 | MCU STM32G474 (Cortex-M4F 170 MHz) |
| REQ-C02 | Saisie de schéma et layout sous Altium Designer |
| REQ-C03 | Firmware C99, bibliothèque de contrôle testable sur PC sans HAL |

## Points ouverts

1. **REQ-P02** — 40 A par moteur ou agrégé (conditionne shunt, gain CSA et cuivre).
2. **Moteur cible** — paramètres réels (Rs, Ls, Kv, p, J). Les documents utilisent un
   moteur de référence.
3. **Encodeurs déportés** — SPI en câble, carte satellite ou sortie ABI.
