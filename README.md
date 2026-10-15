# BLDC-4 — 4-axis FOC controller on STM32G474

*[Version française plus bas](#version-française)*

Control board for **four brushless motors** using field-oriented control
(FOC), commanded over **CAN FD**, with WiFi telemetry (ESP32-C3) kept separate
from the real-time part.

> **Work in progress.** Control firmware, Simulink simulation and Altium
> schematic are done. PCB: power stage routed, signal routing in progress.
> The board has not been manufactured yet: results come from calculations,
> simulations and tests on PC.

## At a glance

- **MCU**: STM32G474 (Cortex-M4F, 170 MHz), 4 × DRV8305 gate drivers, 24 MOSFETs, AS5047P encoders
- **Current loop** at 20 kHz, 1 kHz bandwidth, 63° phase margin
- **Firmware** in C, no HAL in the control code, validated by unit tests
- **Simulink**: the firmware code itself runs against a motor model (software-in-the-loop)
- **PCB**: 6 layers, 190 × 120 mm, designed in Altium Designer

## Preview

PCB in Altium (3D view):

![PCB in Altium, 3D](docs/img/altium_pcb_3d.jpg)

PCB, all layers:

![PCB in Altium, 2D](docs/img/altium_pcb_2d.jpg)

Simulink model: the orange block is the firmware code, compiled and run
against the simulated motor.

![Simulink model](docs/img/simulink_modele.png)

Simulated start-up: offset calibration, rotor alignment, ramp to 3000 rpm,
then a load step.

![Simulated start-up](docs/img/simulink_speed.png)

## Next steps

- [ ] Finish PCB routing, then manufacture the board
- [ ] Low-level STM32 drivers and ESP32 firmware
- [ ] Tests on a real motor

Source code and design files are private, available on request.
The design documents below are in French.

---

## Version française

Carte de commande pour **4 moteurs brushless** en commande vectorielle (FOC),
pilotée par **CAN FD**, avec une télémétrie WiFi (ESP32-C3) séparée de la
partie temps réel.

> **Projet en cours.** Firmware de commande, simulation Simulink et schéma
> Altium terminés. PCB : puissance routée, routage des signaux en cours.
> La carte n'est pas encore fabriquée : les résultats viennent de calculs,
> de simulations et de tests sur PC.

- **MCU** : STM32G474 (Cortex-M4F, 170 MHz), 4 drivers DRV8305, 24 MOSFET, codeurs AS5047P
- **Boucle de courant** à 20 kHz, bande passante 1 kHz, marge de phase 63°
- **Firmware** en C, sans HAL dans la partie commande, validé par des tests unitaires
- **Simulink** : le code du firmware est simulé face à un modèle de moteur (SIL)
- **PCB** 6 couches, 190 × 120 mm, conçu sous Altium Designer

Le code source et les fichiers de conception sont privés, disponibles sur demande.

### Documentation

- [Cahier des charges](docs/00-cahier-des-charges.md)
- [Réglage des boucles](docs/05-reglage-boucles.md) et [architecture du firmware](docs/06-architecture-firmware.md)
- [Conception du PCB](docs/10-conception-pcb.md) et [routage](docs/11-routage-liaisons.md)
- [Modèle Simulink](docs/12-modele-simulink.md)

---

© 2026 Nour Aouini — all rights reserved / tous droits réservés.
