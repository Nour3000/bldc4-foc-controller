# Architecture firmware

## Couches

```
┌──────────────────────────────────────────────────────────────────┐
│ app/          main, ordonnancement, supervision, watchdog CAN    │  cible
├──────────────────────────────────────────────────────────────────┤
│ board/        board_pins.h (généré), init HAL : timers, ADC, DMA │  cible
├────────────────────────────────┬─────────────────────────────────┤
│ foc/  (portable, testé PC)     │ comm/  (portable, testé PC)     │
│  foc_math  Clarke/Park, sincos │  can_proto  consignes CAN FD    │
│  pi        PI + back-calc      │  link       trames IoT          │
│  svpwm     injection min-max   │  cobs       tramage UART        │
│  pll       observateur θ/ω     │  crc16      intégrité           │
│  as5047p   trames encodeur     │                                 │
│  axis      boucle de courant   │                                 │
│            + machine d'états   │                                 │
└────────────────────────────────┴─────────────────────────────────┘
```

**Règle de dépendance** : `foc/` et `comm/` n'incluent aucun en-tête HAL ou
CMSIS. Tout le contrôle est donc exécuté et testé sur PC, sur un modèle de
moteur. La couche `board/` ne fait que transporter des
mesures vers `foc_axis_isr()` et des rapports cycliques vers les registres
`CCRx`.

## Ordonnancement

| Priorité NVIC | Contexte | Période | Travail |
|---|---|---|---|
| 0 | `TIMx_BRK` / HRTIM FLT | événement | PWM déjà coupé en matériel → `foc_axis_raise_fault(DRIVER)` |
| 1 | ADCx `JEOS` (×4, déphasées de 90°) | 50 µs | `foc_axis_isr()` → écriture `CCR1..3` |
| 2 | DMA SPI encodeurs | 50 µs | décodage AS5047P, horodatage de la latence |
| 3 | TIM6 | 1 ms | boucle de vitesse, rampes de consigne |
| 4 | FDCAN1 RX | sporadique | `can_proto_parse()` → file de consignes |
| 5 | USART3 DMA, I2C1 | ~10 ms | télémétrie, TMP117, EEPROM |
| — | boucle principale | — | heartbeat CAN 10 Hz, watchdog de consigne, LED |

Aucune section critique longue : les consignes passent de la priorité 4 à la
priorité 1 par des écritures atomiques de `float` (32 bits alignés sur le
Cortex-M4).

## Chemin d'une période de contrôle (axe M1)

```
t = 0       sommet de porteuse TIM1 (CNT = ARR)
            └─ TRGO → ADC1 injecté : Ia, Ib (634 ns)
            └─ DMA SPI1 : trame AS5047P M1 (2,5 µs)
t ≈ 1 µs    JEOS → ISR ADC1
            foc_axis_isr() :
              offset CSA → PLL (θ, ω) + compensation de latence
              Clarke → Park → 2 PI + découplage → Park inverse (+1,5 Ts)
              SVPWM → rapports cycliques
            écriture CCR (préchargés, pris en compte à l'update)
t ≈ 3 µs    fin d'ISR (~340 cycles estimés)
t = 12,5 µs même séquence pour M2 (entrelacement de 90°)
```

## Machine d'états d'axe

Implémentée dans le module d'axe et testée en boucle fermée sur un modèle de moteur.

```
IDLE ──run──► CALIB_OFFSET ──N échantillons──► ALIGN ──t_align──► RUN
  ▲             │ offset hors tolérance          │                  │
  └─clear──── FAULT ◄────────────────────────────┴──────────────────┘
               (driver, surintensité, Vbus, encodeur, température, timeout CAN)
```

| État | PWM | Rôle |
|---|---|---|
| IDLE | inhibé | attente, consignes nulles |
| CALIB_OFFSET | inhibé | moyenne des sorties CSA à courant nul → offsets |
| ALIGN | actif | `id = I_align` sur `θe = 0` : le rotor s'aligne, le zéro encodeur est mémorisé |
| RUN | actif | boucle de courant sur la consigne `iq` |
| FAULT | inhibé | état sûr verrouillé jusqu'à `clear_faults()` |

Chaque transition remet les consignes à zéro. Ce point a été révélé par les
tests : sans cette remise à zéro, le courant d'alignement survivait au passage
en RUN.

## Robustesse

| Événement | Réponse |
|---|---|
| Trame AS5047P à parité fausse | rejet, extrapolation par la PLL ; défaut après 3 trames consécutives |
| Consigne CAN NaN / ±∞ | rejet dans `can_proto_parse()`, jamais transmise au contrôle |
| Silence CAN > 100 ms en RUN | `AXIS_FAULT_CMD_TIMEOUT` |
| Trame IoT de type « sortant » reçue | rejet `LINK_ERR_FORBIDDEN` : l'IoT ne peut pas commander |
| Redémarrage de l'ESP32 | aucun effet : USART3 en priorité 5, DMA circulaire non bloquant |

## Empreinte

Mesurée avec `arm-none-eabi-gcc 15.2 -O2`, Cortex-M4F :

| Module | .text | .bss |
|---|---|---|
| foc (6 fichiers) | 3 684 o | 2 564 o (table de sinus) |
| comm (4 fichiers) | 848 o | 0 |
| **Total** | **~4,5 ko** | **~2,5 ko** |

Soit moins de 1 % de la Flash (512 ko) et 2 % de la RAM (128 ko).

## Prochaines étapes firmware

1. Projet CubeMX `.ioc` généré depuis le brochage, couche `board/` (HAL LL).
2. Mesure du temps d'ISR réel avec le compteur `DWT->CYCCNT` (contre ~340 cycles
   estimés).
3. Boucle de vitesse et watchdog de consigne.
4. Firmware ESP32-C3 (ESP-IDF) : pont COBS ↔ MQTT/TLS.
