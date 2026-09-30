# PROJECT STATE — MCU V1 / GPIO wake-suspend / DSI-FPC prototype

**Date :** 2026-09-30  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** addendum maître — décisions V1 prises + validations encore requises  
**Base power :** `PROJECT_STATE_POWER_USB_C_V1_2026-09-28.md`  
**Base STR :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-24_MASTER_STR_FIRST_DEEP_OK.md`  
**Base écran :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-23_MASTER_SCREEN_PROTO_READY.md`  
**État modem :** `ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md`

Ce document fige uniquement les décisions prises pour le **prototype V1 MCU** et la liaison **Q6A J10 ↔ interposer J1**. Il ne remplace pas l'architecture finale de la carte power : le MCU final reste un superviseur always-on et la carte finale devra toujours permettre les vrais power-gates Q6A/modem prévus dans le document power du 28/09.

---

# 1. MCU de prototype

[FIGÉ POUR LE PROTOTYPE]

Le MCU de développement est un **RP2040-Tiny** basé sur RP2040.

Pendant le développement :

```text
PC ── USB ──> RP2040-Tiny
```

Cette liaison USB PC est **temporaire** et sert à :

- programmer le MCU ;
- modifier `main.py` ;
- lire le REPL / logs ;
- permettre à un agent de développement de tester et déboguer le firmware.

Elle ne représente pas l'alimentation finale du superviseur. Dans le téléphone final, le MCU devra être alimenté par le domaine always-on de la carte power.

## 1.1 Firmware de prototype

[ACTUEL / PROTOTYPE]

MicroPython est retenu pour la phase de développement, car il permet une itération rapide par USB/REPL et un `main.py` autonome.

Ce choix ne fige pas le firmware final. Une migration ultérieure vers le Pico SDK / C/C++ reste possible si nécessaire pour réduire la consommation et obtenir un contrôle plus fin des modes basse consommation.

---

# 2. Firmware de caractérisation RP2040-Tiny déjà testé

Le firmware de test `main.py` exécute une séquence cyclique de cinq états d'environ 10 s chacun :

```text
1. Idle
   - machine.idle() en boucle
   - LED verte fixe

2. Actif
   - charge CPU
   - clignotement rouge rapide

3. Sleep
   - lightsleep(970 ms)
   - flash bleu bref environ 1 fois/s

4. Idle opt
   - lecture ADC température interne
   - fréquence CPU abaissée à 48 MHz entre les réveils
   - lightsleep entre activités
   - flash jaune

5. Deep sleep
   - deepsleep(10 s)
   - LED éteinte
   - redémarrage matériel/logique en fin de cycle selon l'implémentation MicroPython RP2
```

Un heartbeat indépendant a été ajouté :

```text
GP2 = sortie GPIO simple de signal de vie
```

Il permet de confirmer l'exécution du firmware indépendamment de la WS2812 embarquée sur GP16, dont le fonctionnement cesse à une tension plus élevée que le cœur RP2040.

---

# 3. Mesures d'alimentation RP2040-Tiny

[CONFIRMÉ PAR MESURES PROTOTYPE]

Deux configurations ont été essayées :

```text
A. alimentation par VSYS variable
B. alimentation directe du rail 3V3, bypassant l'étage VSYS -> 3V3
```

Résultats observés :

```text
Idle / Actif       ~20 mA
Sleep / Idle opt   ~1 à 2 mA
```

À état donné, le courant mesuré est pratiquement identique entre alimentation VSYS et alimentation directe 3V3 sur la plage testée.

Avec VSYS variable, le MCU a continué à exécuter le heartbeat jusqu'à environ :

```text
VSYS ~1,6 V
```

puis décroche en dessous.

**Cette valeur est une observation expérimentale, pas une tension nominale à retenir pour le produit.** Elle ne valide pas les niveaux GPIO, la flash, l'USB ou les autres périphériques hors spécifications constructeur.

## 3.1 Interprétation V1

Pour le prototype téléphone :

- `machine.idle()` n'apporte pas de gain significatif dans les mesures actuelles ;
- `lightsleep()` est le mode MicroPython utile pour réduire fortement la consommation ;
- le mode `deepsleep()` MicroPython RP2 ne doit pas être interprété comme la preuve d'un état matériel radicalement plus profond sans mesure dédiée ;
- les consommations de la carte de développement incluent ses composants auxiliaires et ne représentent pas la consommation atteignable par un MCU final optimisé.

---

# 4. Liaison MCU ↔ Q6A — simplification V1

[FIGÉ POUR V1]

Contrainte mécanique du prototype : seulement **3 fils simples** entre le Q6A et le RP2040-Tiny.

La V1 retient :

| Fil | Q6A header 40 pins | Signal Q6A | Rôle |
|---|---:|---|---|
| 1 | **34** | GND | masse commune |
| 2 | **36** | GPIO59 | `MCU_WAKE` |
| 3 | **37** | GPIO58 | `MCU_SLEEP_REQ` |

Source pinout officielle Radxa :

```text
pin 34 = GND
pin 36 = GPIO_59 / UART14_RX / SPI14_CS_0
pin 37 = GPIO_58 / UART14_TX / SPI14_SCLK
```

Référence : `https://docs.radxa.com/en/dragon/q6a/hardware-use/pin-gpio`

## 4.1 Masse commune

[OBLIGATOIRE]

Même si le Pico est temporairement alimenté par le PC et le Q6A par une autre source :

```text
GND Pico ↔ GND Q6A
```

est obligatoire pour les GPIO directs.

Ne pas relier les rails 5 V/3V3 des deux systèmes entre eux dans ce prototype uniquement parce que les masses sont communes.

## 4.2 Pas de 4N35 en V1

[FIGÉ POUR V1]

Le 4N35 avait été envisagé pour simuler l'entrée matérielle `PWR` du Q6A. Cette option est abandonnée pour la première passe afin de réduire l'électronique du prototype.

La V1 utilise **uniquement les GPIO du header Q6A**.

Conséquence importante :

```text
V1 GPIO-only = suspend + wake à valider
V1 GPIO-only != vrai power-on depuis Q6A physiquement OFF
V1 GPIO-only != hard power-cycle
```

Le vrai ON/OFF complet restera une fonction de la future carte power via `MAIN_PWR` et/ou l'entrée PWR matérielle appropriée.

---

# 5. GPIO59 — ligne WAKE

[CIBLE V1 / À VALIDER SUR LE Q6A RÉEL]

GPIO59 est choisi comme ligne de réveil car :

1. il est exposé sur le header physique pin 36 ;
2. dans le pinctrl Qualcomm SC7280, le mapping PDC contient explicitement :

```text
GPIO59 -> PDC wake IRQ 110
```

Référence noyau : `drivers/pinctrl/qcom/pinctrl-sc7280.c`, table `sc7280_pdc_map`.

Cela en fait un bon candidat pour une source de wake profonde, mais **la capacité réelle sur notre BSP/Q6A doit encore être validée** avec le DTBO exact et un vrai cycle STR.

Configuration logicielle envisagée :

```text
GPIO59
  -> entrée Q6A
  -> IRQ
  -> wakeup-source
  -> éventuellement gpio-keys / KEY_POWER
```

Le choix final entre un `gpio-keys` générant `KEY_POWER` et un petit driver/handler dédié reste ouvert jusqu'au test.

Critère de validation :

```text
Q6A en mem_sleep=deep
Pico émet l'impulsion MCU_WAKE
Q6A reprend correctement
wake IRQ identifiée
Android opérationnel après resume
```

---

# 6. GPIO58 — demande SLEEP

[CIBLE V1]

GPIO58, header pin 37, est réservé à :

```text
MCU_SLEEP_REQ
```

Le Pico ne doit pas forcer directement un état électrique de sommeil ni couper l'alimentation du Q6A.

Séquence cible :

```text
Pico
  |
  +--> impulsion MCU_SLEEP_REQ
          |
          v
Q6A driver/service
          |
          v
Android/Linux prépare le suspend
          |
          v
Suspend-to-RAM deep
```

La logique Q6A exacte reste à implémenter et tester.

GPIO58 n'est pas retenu comme source de wake ; le réveil V1 reste dédié à GPIO59.

---

# 7. États que le MCU V1 peut et ne peut pas piloter

| État / fonction | V1 GPIO-only |
|---|---|
| demander un suspend Android | **OUI, cible** |
| réveiller depuis STR `deep` | **OUI, cible à valider** |
| gérer un pseudo-bouton Power logiciel | **possible via GPIO59 / gpio-keys, à valider** |
| savoir automatiquement que le shutdown est terminé | **NON avec ces 3 fils** |
| couper physiquement le Q6A | **NON** |
| rallumer après coupure physique totale | **NON** |
| hard power-cycle | **NON** |

Cette limitation est volontaire pour le proto. L'architecture finale définie dans `PROJECT_STATE_POWER_USB_C_V1_2026-09-28.md` conserve la commande physique `MAIN_PWR` par le MCU.

---

# 8. Modem EC25 ↔ MCU — architecture préparée mais non câblée dans ces 3 fils

La liaison principale Android ↔ EC25 reste :

```text
Q6A USB2 Host -> EC25 core board
```

État actuellement validé : le modem GA répond, le RIL AIDL1 expose les sept services Radio et Android atteint `SIM ABSENT`. Voir `ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md`.

Pour la future supervision always-on, la cible reste :

```text
EC25 RI  -> MCU input/IRQ
MCU DTR  -> EC25
MCU UART <-> EC25
```

`RI` est le signal naturel pour avertir le MCU qu'un événement/URC est disponible. Le MCU devra ensuite interpréter l'événement, typiquement via UART, avant de décider s'il faut réveiller le Q6A.

**Ne pas câbler directement ces signaux au RP2040 avant validation électrique de la core-board.** Le carrier expose `VIO`, `RI`, `DTR`, `RXD`, `TXD`, mais le niveau réel à utiliser doit être mesuré/confirmé avant connexion.

---

# 9. Q6A J10 ↔ interposer J1 — validation du pinout

[VALIDÉ PAR COMPARAISON SCHÉMA OFFICIEL + `AdaptateurEcran(7)`]

Source Q6A : schéma officiel Radxa Dragon Q6A V1.21, feuille `LCD_MIPI_EDP`, J10 `LCD_39P`.

Fichier prototype inspecté :

```text
AdaptateurEcran(7).zip
repo interne AdaptateurEcran
HEAD observé : 8c28084
message : chore(fab): regenerate JLCPCB production files with the M2 holes
```

Footprint J1 réel dans `V1/V1.kicad_pcb` :

```text
interposer_q6a:Hirose_FH35C-39S-0.3SHW_1x39_P0.3mm
MPN : FH35C-39S-0.3SHW(50)
39 contacts
pitch électrique 0,3 mm
```

## 9.1 Comparaison pin par pin

Le mapping constaté dans le PCB correspond au J10 officiel :

```text
 1  VDD3V3             -> IO_3V3
 2  IOVCC1V8-3V3       -> IO_1V8
 3  SENSOR-INT          -> SENSOR_INT (NC volontaire)
 4  RESET               -> DISP_RST_N
 5  NC                  -> NC
 6  GND                 -> GND
 7  MIPI-0N             -> DSI_D0_N
 8  MIPI-0P             -> DSI_D0_P
 9  GND                 -> GND
10  MIPI-1N             -> DSI_D1_N
11  MIPI-1P             -> DSI_D1_P
12  GND                 -> GND
13  MIPI-CKN            -> DSI_CLK_N
14  MIPI-CKP            -> DSI_CLK_P
15  GND                 -> GND
16  MIPI-2N             -> DSI_D2_N
17  MIPI-2P             -> DSI_D2_P
18  GND                 -> GND
19  MIPI-3N             -> DSI_D3_N
20  MIPI-3P             -> DSI_D3_P
21  GND                 -> GND
22  GND                 -> GND
23  TP-RESET            -> TP_RESET
24  TP-VCC              -> TP_VCC
25  TP-INT              -> TP_INT
26  TP-SDA              -> TP_SDA
27  TP-SCL              -> TP_SCL
28  GND                 -> GND
29  GND                 -> GND
30  VCC3V31             -> LCD_3V3
31  VCC3V32             -> LCD_3V3
32  GND                 -> GND
33  GND                 -> GND
34  LED-1               -> LED_K (NC volontaire)
35  LED-                -> LED_K (NC volontaire)
36  NC                  -> NC
37  NC                  -> NC
38  LED+1               -> LED_A (NC volontaire)
39  LED+                -> LED_A (NC volontaire)
```

Les pins backlight J10 34/35/38/39 sont volontairement non utilisées par J1 dans le chemin principal actuel, car l'interposer emploie son propre TPS61194.

Conclusion :

```text
ordre J10 Q6A -> J1 interposer : 1:1
lanes DSI                  : conformes
polarités N/P              : conformes
GND intercalés             : conformes
```

---

# 10. Connecteur et nappe FPC J10 ↔ J1

Le Q6A utilise un connecteur MIPI DSI :

```text
39 pins
pitch 0,3 mm
famille Hirose FH35C
```

La série FH35C est un connecteur FPC **dual-sided / Top & Bottom Contact** et accepte un FPC d'environ **0,2 mm d'épaisseur**.

Références publiques :

```text
https://docs.radxa.com/en/dragon/q6a/hardware-use/mipi-dsi
https://www.hirose.com/product/series/FH35C
```

## 10.1 Nappe cible V1

[Achat prototype]

```text
39 contacts
pitch 0,3 mm
FPC compatible ~0,2 mm
longueur typique 60 à 100 mm selon mécanique
```

Le vendeur examiné propose deux variantes nommées :

```text
A-Forward Direction
B-Opposite Direction
```

La variante **A-Forward** est actuellement la cible d'achat pour obtenir une liaison droite 1:1 entre les deux connecteurs.

**Important : `A-Forward` / `B-Opposite` est une nomenclature vendeur et ne constitue pas une norme universelle.** Ne jamais mettre le système sous tension en se fondant uniquement sur le nom commercial.

## 10.2 Validation obligatoire de la nappe avant alimentation

Avec la nappe insérée hors tension, vérifier au minimum :

```text
Q6A J10 pin 1  <-> interposer J1 pin 1
Q6A J10 pin 2  <-> interposer J1 pin 2
Q6A J10 pin 39 <-> interposer J1 pin 39
```

Puis idéalement contrôler quelques masses et une paire DSI supplémentaire.

Si le test donne `1 -> 39`, ne pas alimenter : la nappe/orientation n'est pas la bonne.

---

# 11. Ce qui est décidé / ce qui reste à prouver

## Décidé

```text
- RP2040-Tiny pour prototype MCU
- MicroPython pour développement rapide
- USB PC temporaire pour programmation/debug agent
- 3 fils MCU <-> Q6A seulement en V1
- pin 34 GND
- pin 36 / GPIO59 = MCU_WAKE
- pin 37 / GPIO58 = MCU_SLEEP_REQ
- aucun 4N35 dans cette première passe
- vrai hard OFF/ON reporté à la carte power
- RI EC25 destiné au MCU dans l'architecture future
- J1 interposer suit le pinout J10 Q6A 1:1
- nappe 39P / 0,3 mm / ~0,2 mm
- A-Forward comme variante commerciale visée, sous réserve de continuité réelle
```

## À valider

```text
[ ] DTBO/pinctrl GPIO59 en entrée wake
[ ] gpio-keys/handler exact pour MCU_WAKE
[ ] vrai réveil depuis mem_sleep=deep par GPIO59
[ ] handler/service de MCU_SLEEP_REQ sur GPIO58
[ ] cycle suspend -> wake répété
[ ] absence de régression STR/USB
[ ] niveaux électriques RI/DTR/UART de la core-board EC25
[ ] nappe FPC réelle : continuité 1->1 / 2->2 / 39->39
[ ] fonctionnement écran après fabrication, bring-up 60 Hz d'abord
```

---

# 12. Mission suivante recommandée — bring-up MCU V1

Ordre strict :

```text
1. Pico seul : firmware simple + heartbeat GP2 + logs USB
2. relier uniquement GND commun
3. relier GPIO58/GPIO59, Q6A configuré en entrées sûres
4. vérifier niveaux au multimètre avant toute action
5. valider MCU_SLEEP_REQ sans entrer en deep
6. valider ensuite un vrai STR deep
7. valider MCU_WAKE par GPIO59
8. répéter plusieurs cycles suspend/wake
9. seulement ensuite ajouter la supervision modem RI/UART
```

Le critère V1 n'est pas encore l'extinction matérielle complète. Le critère est d'obtenir une boucle reproductible :

```text
MCU -> demande suspend propre -> Q6A deep -> MCU wake -> reprise Android correcte
```
