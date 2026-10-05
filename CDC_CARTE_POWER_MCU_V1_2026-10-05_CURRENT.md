# CDC — Carte Power / MCU V1 du Maker Phone Q6A — CURRENT

**Date :** 2026-10-05  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Révision :** **V0.9 — architecture courante + BOM consolidée**  
**Fabrication cible :** JLCPCB Economic PCBA  
**Priorité composants :** Basic > Promotional > Extended ; stock, prix et classification à revalider au moment de la commande.

Ce document est la **référence électrique normative courante** de la carte Power/MCU. Il supersède les CDC Power/MCU antérieurs dès qu'ils sont contradictoires. La V0.8 du 2026-10-03/05 reste historique/audit.

La modification principale V0.9 est l'abandon de l'alimentation EC25 BAT directement depuis SYS : le modem reçoit désormais un rail régulé d'environ **3.825 V** produit par un **RT6154AGQW**. La tension haute de la JK50 n'est donc plus appliquée directement au BAT du EC25.

---

# 0. Architecture normative

```text
                        Motorola JK50 1S
                       Molex 5050040812
                              |
                  J_BAT = Molex 5050060812
                              |
                          BAT_RAW+
                              |
                BAT_ARM : interrupteur SPST externe
                 via 2 grosses pastilles THT
                              |
                             BAT
                              |
USB-C extérieur 5 V           |
DEVICE uniquement             |
        |                     |
        +--> VBUS --> TVS --> BQ25628E --> SYS ------------------------------+
        |                         |                                           |
        |                         +<------> BAT                                |
        |                                                                     |
        +--> D+/D- -----------------------------> Q6A USB OTG DEVICE          |
        +--> VBUS -- Schottky ------------------> Q6A OTG VBUS                |
                                                                              |
SYS --------------------------------------------------------------------------+
 |                                                                             |
 +--> TPS610995 --> MCU_VSYS --> VSYS RP2040-Tiny                              |
 |       ^                                                                     |
 |       +-- EN <- STORAGE_SW ; EN pulldown 100 kOhm                          |
 |                                                                             |
 +--> MAIN_PWR : JMTQ55P02A --> Q6A J19                                       |
 |       commande AO3400A + maintien RC ~1 s                                  |
 |                                                                             |
 +--> MODEM_PWR : JMTQ55P02A --> MODEM_RAW                                    |
 |       commande AO3400A + maintien RC ~1 s                                  |
 |                              |                                              |
 |                              +--> RT6154AGQW buck-boost                     |
 |                                      |                                      |
 |                                      +--> EC25_BAT_3V8 ~3.825 V --> EC25 BAT|
 |                                                                             |
 +--> ANNEXE1 : AO3401A + driver AO3400A -> ampli HP                          |
 +--> ANNEXE2 : AO3401A + driver AO3400A -> annexes                           |

Q6A USB2 HOST #1 VBUS ----> carte Power ----> switch AO3401A/AO3400A ----> EC25 USB_VBUS
Q6A USB2 HOST #1 D+/D- ----------------------------------------------------> EC25 DP/DN direct
Q6A USB2 HOST #1 GND ------------------------------------------------------> EC25 GND

Q6A USB2 HOST #2 ----------------------------------------------------------> caméra USB future
Q6A USB2 HOST #3 ----------------------------------------------------------> réserve / futur USB utilisateur

RP2040-Tiny
  <-> BQ25628E I2C ; états/faults par polling
  <-> Q6A SBS/I2C
   -> Q6A PWR_ON_KEY via S8050
   -> Q6A SHUTDOWN_REQ via GPIO58
   <- Q6A HEARTBEAT_STATE via GPIO59
  <-> EC25 UART/DTR/RI via SN74AVC4T245
   -> EC25 PWRKEY via S8050
   -> MAIN_PWR / MODEM_PWR / EC25_USB_VBUS / ANNEXE1 / ANNEXE2
   <- bouton Power utilisateur
   <- CC_SENSE analogique fusionné
   <- option NTC2 JK50 sur GP28/ADC2, DNP jusqu'à caractérisation

Boutons Volume
  VOL+ -> Q6A GPIO31 direct
  VOL- -> Q6A GPIO30 direct
  aucune liaison vers RP2040
```

La carte Power ne transporte ni DSI écran ni tactile. Pour l'USB Q6A <-> EC25, seul VBUS 5 V traverse la carte afin d'être commutable ; D+/D- restent en faisceau court direct Q6A <-> carrier EC25.

---

# 1. Batterie JK50, connecteur, NTC et BAT_ARM

## 1.1 Batterie

```text
Référence famille       : Motorola JK50, Genuine Service Pack de préférence
Architecture             : pack smartphone 1S protégé / PCM interne
Tension nominale        : ~3.80 V
Capacité rated          : ~4850 mAh
Capacité typical        : ~5000 mAh
Énergie                 : ~18.5 à 19 Wh selon révision
Tension pack admissible : jusqu'à ~4.40 V pour la cellule d'origine
Charge max documentée   : ~3.0 A
Décharge max documentée : ~4.85 A
Dimensions de travail   : ~86.1 x 65.0 x 4.8 mm
Réservation prototype   : >=87 x 66 x 5.5 mm
```

La vraie batterie reçue doit être mesurée avant gel mécanique. Le PCB est compatible avec une politique de charge haute tension de la JK50 puisque l'EC25 est désormais isolé par le RT6154A ; **la valeur firmware finale de VREG reste à valider sur le pack réel**. Une valeur conservatrice peut être utilisée pendant le bring-up.

Le PCM de la JK50 reste la protection ultime de sous-tension/surintensité du pack. Le MCU doit toutefois demander un shutdown propre du Q6A/modem avant d'atteindre la coupure PCM ; les seuils exacts relèvent du firmware et n'ajoutent aucun composant PCB.

## 1.2 Connecteur batterie

```text
PCB                    : Molex 5050060812 / LCSC C779875
Mate flex batterie     : Molex 5050040812
Pins 1 + 8             : GND / BATT-
Pins 4 + 5             : VBAT+
Pin 2                  : ID2
Pin 3                  : ID1
Pin 6                  : NTC1
Pin 7                  : NTC2
```

Le footprint et l'orientation proviennent exclusivement du drawing Molex officiel. ID1/ID2 restent haute impédance + testpoints en V1.

## 1.3 NTC1 / NTC2

Le BQ25628E ne reçoit qu'une mesure TS à la fois. Le schéma doit conserver le choix physique par straps DNP :

```text
NTC1 -> TP -> strap 0 Ohm/DNP -> BQ_TS
NTC2 -> TP -> strap 0 Ohm/DNP -> BQ_TS
```

Une seule de ces deux branches peut être peuplée vers BQ_TS.

En plus, NTC2 est pré-routé vers le RP2040 afin de permettre une seconde mesure thermique logicielle :

```text
NTC2 -> TP -> R_SER_ADC DNP -> GP28 / ADC2
3V3_MCU -> R_BIAS_NTC2 DNP -> NTC2
GP28/ADC2 -> C_ADC_NTC2 DNP -> GND
```

Valeurs de R_SER/R_BIAS/C_ADC non figées avant caractérisation du pack. Baseline fabrication : **DNP**. L'option `NTC2 -> GP28/ADC2` est mutuellement exclusive avec l'option `GP28 -> EC25 RESET_N open-drain`.

La courbe R/T réelle de NTC1/NTC2 doit être mesurée avant de figer le réseau TS/JEITA. Le comportement attendu est celui d'un téléphone classique : la sécurité de charge matérielle reste assurée par le BQ, tandis que le MCU peut appliquer des limites supplémentaires si la seconde température est disponible.

## 1.4 BAT_ARM

```text
J_BAT pins4/5 -> BAT_RAW+ -> PAD_BAT_ARM_A
                                  |
                          interrupteur externe
                                  |
                           PAD_BAT_ARM_B -> BAT -> BQ BAT
```

Exigences : SPST DC basse tension, >=5 A, faible résistance de contact, 2 grosses pastilles THT, cuivre BAT dimensionné pour le pack.

`BAT_ARM` isole la batterie mais pas l'USB-C : USB-C présent peut toujours alimenter SYS via le BQ.

## 1.5 Protection pack

```text
- pas de second BMS/PCM sur le PCB ;
- BQ25628E = chargeur/power-path, pas PCM batterie ;
- pas de fusible obligatoire baseline V1 ;
- pas de PTC obligatoire ;
- pas de Schottky série BAT ;
- pas de MOS anti-inversion série baseline.
```

---

# 2. BQ25628E — chargeur / power-path

## 2.1 Référence et câblage

```text
U_CHG       : BQ25628ERYKR / C18221178
Batterie    : 1S
Power-path  : NVDC
ADC/I2C     : oui
CE          : GND
SDA         : GP4 + pull-up 10 kOhm vers 3V3_MCU
SCL         : GP5 + pull-up 10 kOhm vers 3V3_MCU
INT         : non relié au MCU ; TP optionnel seulement
PG          : testpoint
STAT        : NC
QON         : testpoint / fonction interne conservée
```

Le MCU surveille le BQ par polling I2C. La sécurité de charge ne dépend pas du RP2040.

## 2.2 Cold boot obligatoire

```text
1. GPIO dans leurs états sûrs : MAIN/MODEM/USB_VBUS/ANNEXES OFF.
2. Vérifier ACK I2C du BQ.
3. Désactiver explicitement le watchdog BQ sauf politique ultérieure documentée.
4. Programmer VREG, VSYSMIN, ICHG, IINDPM/ILIM, EN_EXTILIM, ADC, TS/thermique.
5. Relire les registres critiques.
6. Vérifier faults.
7. En cas d'échec : rester OFF/FAULT.
8. Si valide : autoriser MAIN_PWR puis la séquence Q6A.
```

Valeurs de départ :

```text
RILIM               : 5.6 kOhm / C23189 Basic / 1 %
VSYSMIN             : 3.84 V cible
source inconnue     : IINDPM conservateur ~0.44 A
Rp 1.5 A reconnu    : ~1.35 A max de départ
ICHG                : <=2 A
VREG                : paramètre firmware à revalider JK50 ; plus limité par EC25 direct
```

`IBAT_PK`, `TREG`, `ITERM` et la courbe TS finale restent à figer après mesures thermiques.

## 2.3 Reset MCU pendant RUN

Les RC maintiennent temporairement MAIN_PWR, MODEM_PWR et EC25_USB_VBUS. Au début du firmware, GP10/GP29/GP6 doivent être réaffirmés avant expiration du hold, puis le BQ et les états Q6A/modem sont resynchronisés.

## 2.4 Passifs BQ

```text
L_CHG    : XRIM252012S1R0MBCA / C22471110 / 1 uH
CVBUS    : 1 x 1 uF / C52923
CVBUS_HF : 1 x 100 nF / C1525
CPMID    : 2 x 10 uF / C15850
CPMID_HF : 1 x 100 nF / C1525
CSYS     : 3 x 10 uF / C15850
CBAT     : 1 x 1 uF / C52923
CREGN    : 1 x 4.7 uF / C1779
CBTST    : 1 x 47 nF 50 V / C1622
```

Layout strict selon TI.

---

# 3. USB-C extérieur — charge + DEVICE uniquement

```text
Fonctions : charge 5 V, ADB, EDL, MTP/USB device
Pas de host USB-C, pas de DRP, pas de PD V1.
```

```text
TYPE-C-31-M-12 / C165948
A6+B6 -> D+ -> SRV05-4 -> Q6A OTG D+
A7+B7 -> D- -> SRV05-4 -> Q6A OTG D-
VBUS  -> SMF5.0A -> BQ VBUS
VBUS  -> B5819W SL -> Q6A OTG VBUS
CC1   -> 5.1 kOhm -> GND
CC2   -> 5.1 kOhm -> GND
CC1 -> 470 kOhm --+
                    +--> CC_SENSE -> GP26/ADC0
CC2 -> 470 kOhm --+
CC_SENSE -> 100 nF -> GND
```

D+/D- : paire USB2 ~90 Ohm différentiel sur stack-up réel, plan GND continu, aucun stub inutile.

---

# 4. RP2040-Tiny / always-on / Storage OFF

```text
Module : Waveshare RP2040-Tiny soudé manuellement BOTTOM, sans headers
SYS -> TPS610995DRVR / C2071098 -> MCU_VSYS -> VSYS RP2040-Tiny
L_MCU : MAKK2016T2R2M / C92923 / 2.2 uH
CIN   : 10 uF
COUT  : 2 x 10 uF
```

```text
SYS -> STORAGE_SW -> EN TPS610995
EN -> 100 kOhm -> GND
fermé  : MCU alimenté
ouvert : hard-off MCU / stockage
```

## 4.1 Pinout RP2040-Tiny V0.9

```text
GP0   -> EC25 RXD via SN74AVC4T245       UART TX MCU
GP1   <- EC25 TXD via SN74AVC4T245       UART RX MCU
GP2   -> EC25 DTR via SN74AVC4T245
GP3   <- EC25 RI via SN74AVC4T245
GP4   <-> BQ SDA
GP5   ->  BQ SCL
GP6   ->  EC25_USB_VBUS_EN
GP7   <-  PWR_BUTTON utilisateur, actif bas
GP8   ->  Q6A PWR_ON_KEY via S8050
GP9   ->  Q6A SHUTDOWN_REQ, actif bas / Hi-Z au repos
GP10  ->  MAIN_PWR_EN avec RC hold
GP11  ->  ANNEXE1_EN
GP12  ->  ANNEXE2_EN
GP13  ->  EC25 PWRKEY via S8050
GP14  <-> Q6A SBS_SDA
GP15  <-> Q6A SBS_SCL
GP16  : WS2812 du module ; aucune fonction système obligatoire
GP26  <-  USB_CC_SENSE / ADC0
GP27  <-  Q6A HEARTBEAT_STATE / GPIO59
GP28  <-  option NTC2_ADC / ADC2 DNP
       ->  TP_GP28_3V3 direct
       ->  option EC25 RESET_N open-drain DNP, exclusive avec NTC2_ADC
GP29  ->  MODEM_PWR_EN avec RC hold
```

---

# 5. MAIN_PWR / MODEM_PWR / annexes

## 5.1 MAIN_PWR et MODEM_PWR

```text
Q_MAIN_P  : JMTQ55P02A / C2890429
Q_MODEM_P : JMTQ55P02A / C2890429
```

Par rail :

```text
SYS -> source JMTQ55P02A
P-MOS gate -> 100 kOhm -> SYS
P-MOS gate -> drain AO3400A
AO3400A source -> GND
GPIO -> 10 kOhm -> gate AO3400A
AO gate -> 1 MOhm -> GND
AO gate -> 1 uF -> GND
```

```text
GPIO HIGH -> ON
GPIO LOW  -> OFF rapide
GPIO Hi-Z -> maintien temporaire
cible hold mesurée : ~0.7 à 1.5 s
```

`MODEM_PWR_OUT` de V0.8 devient `MODEM_RAW` en V0.9 : ce rail alimente l'entrée du RT6154A, pas directement EC25 BAT.

## 5.2 ANNEXE1 / ANNEXE2

Pour chaque annexe :

```text
SYS -> source AO3401A / C15127
AO3401A drain -> ANNEXE_OUT
AO3401A gate -> 100 kOhm -> SYS
AO3401A gate -> drain AO3400A / C20917
AO3400A source -> GND
GP11/GP12 -> 10 kOhm -> gate AO3400A
gate AO3400A -> 100 kOhm -> GND
aucun condensateur de hold
```

```text
ANNEXE1 : alimentation ampli HP, cible locale ~1 A
ANNEXE2 : SYS commuté générique, cible locale ~1 A
```

---

# 6. Q6A <-> carte Power / MCU

```text
Q6A J20 pin 36 / GPIO59 -> HEARTBEAT_STATE -> GP27
Q6A J20 pin 37 / GPIO58 <- SHUTDOWN_REQ <- GP9
Q6A J20 pin 3 / GPIO24 / I2C6_SDA <-> GP14 SBS_SDA
Q6A J20 pin 5 / GPIO25 / I2C6_SCL <-> GP15 SBS_SCL
Q6A J20 pin 1 ou 17 / 3V3 -> Q6A_3V3_REF
Q6A PWR_ON_KEY <- S8050 <- GP8
MAIN_PWR_OUT -> Q6A J19+
GND -> Q6A J19-
```

J19 cible :

```text
R7        -> DNP/retiré
R24       -> 2 mOhm 1 %
R190      -> 100 kOhm GND
R191      -> 10 kOhm GND
FB4       -> DNP
R185..189 -> DNP si inspection confirme
```

### PWR_ON_KEY

```text
GP8 -> 10 kOhm -> base S8050
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

### SHUTDOWN_REQ

```text
Q6A_3V3 -> 100 kOhm -> SHUTDOWN_REQ
SHUTDOWN_REQ -> 10 kOhm série -> GP9
MCU actif : tire LOW
repos : GP9 input / Hi-Z
```

Cette ligne demande uniquement une extinction Android propre ; elle ne force pas le suspend normal.

### HEARTBEAT_STATE

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Sémantique minimale : RUN / SHUTDOWN_IN_PROGRESS / SHUTDOWN_READY / AUTO_OFF_ALLOWED. L'absence de heartbeat seule ne provoque jamais une coupure immédiate.

### SBS

```text
GP14 <-> Q6A I2C6_SDA
GP15 <-> Q6A I2C6_SCL
2 x 4.7 kOhm vers Q6A_3V3 : DNP par défaut
```

Ne jamais tirer SBS vers 3V3_MCU.

---

# 7. Boutons utilisateur

```text
POWER : bouton NO vers GND -> GP7 ; pull-up 10 kOhm vers 3V3_MCU
VOL+  : Q6A J20 pin 29 / GPIO31 direct
VOL-  : Q6A J20 pin 32 / GPIO30 direct
```

Appui Power court : MCU transmet un pulse PWR_ON_KEY. Android reste maître de l'écran, des wakelocks et du suspend. Appui long : SHUTDOWN_REQ d'abord, long PWR_ON_KEY puis hard cut seulement en dernier recours.

---

# 8. EC25 — alimentation régulée, USB et supervision

## 8.1 Rail BAT régulé — RT6154A

Architecture normative V0.9 :

```text
SYS
 |
 +--> JMTQ55P02A MODEM_PWR
          |
       MODEM_RAW
          |
          +--> RT6154AGQW / C250400
                    |
                    +--> EC25_BAT_3V8 ~= 3.825 V
                                  |
                                  +--> carrier EC25 BAT
```

Le JMTQ55P02A est conservé : il fournit une coupure primaire indépendante, le hold pendant reset MCU et un hard-off clair. Le RT6154A fournit la régulation fine du BAT modem et découple l'EC25 de la tension JK50.

### RT6154A — câblage normatif

```text
U_MODEM_REG : RT6154AGQW / C250400 / Extended
L_MODEM     : FTC322520S2R2MBCA / C5832397 / 2.2 uH / Extended
VIN/VINA    : MODEM_RAW
EN          : MODEM_RAW ; ne jamais laisser flottant
PS/SYNC     : GND ; Power-Save Mode autorisé à faible charge
PGOOD       : TP_MODEM_PGOOD ; pull-up DNP seulement si nécessaire
FB          : pont vers VOUT/GND
CIN         : 2 x 10 uF + 100 nF près de VIN/VINA
COUT        : 3 x 47 uF 10 V + 100 nF près de VOUT
EP/PGND     : grand cuivre GND + vias thermiques
```

Feedback :

```text
R_TOP = 1.00 MOhm + 330 kOhm en série = 1.33 MOhm
R_BOT = 200 kOhm
VFB   = 0.500 V typ.
VOUT  = VFB x (1 + R_TOP/R_BOT)
      ~= 0.5 x (1 + 1.33M/200k)
      ~= 3.825 V
```

Tolérance réelle à vérifier avec tolérances résistances + VFB. La tension doit rester dans la plage BAT EC25 dans tous les états batterie/USB/température.

Le RT6154A est un buck-boost synchrone : il régule correctement lorsque SYS est au-dessus ou en dessous de ~3.825 V. Il dispose d'un load disconnect à l'arrêt ; malgré cela le hard-off final exige MODEM_PWR OFF **et** EC25_USB_VBUS OFF, puis validation du back-power via D+/D-.

## 8.2 USB Q6A -> EC25

D+/D- directs :

```text
Q6A USB2 host D+ ---------------------------------> EC25 DP
Q6A USB2 host D- ---------------------------------> EC25 DN
Q6A GND ------------------------------------------> EC25 GND
```

VBUS commutable :

```text
Q6A USB2 HOST VBUS
 -> PAD_Q6A_EC25_VBUS_IN
 -> AO3401A high-side
 -> PAD_EC25_USB_VBUS_OUT
 -> EC25 USB_VBUS
```

Commande :

```text
AO3401A gate -> 100 kOhm -> source 5 V
AO3401A gate -> drain AO3400A
AO3400A source -> GND
GP6 -> 10 kOhm -> gate AO3400A
AO gate -> 1 MOhm -> GND
AO gate -> 1 uF -> GND
```

## 8.3 UART / DTR / RI

```text
SN74AVC4T245DR / C22495
VCCA = EC25 VIO (~1.8 V)
VCCB = 3V3_MCU
EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

DIR/OE doivent être câblés selon les deux groupes du composant avec états définis et isolation lorsque le domaine opposé est éteint. 100 nF sur chaque alimentation. Le détail DIR/OE doit être validé au schéma avant fabrication ; il ne doit pas créer de back-power.

## 8.4 PWRKEY / RESET

```text
GP13 -> 10 kOhm -> base S8050 -> EC25 PWRKEY open-collector
base -> 100 kOhm -> GND
```

Recovery : AT propre -> PWRKEY -> USB_VBUS OFF + MODEM_PWR OFF.

Option DNP RESET_N :

```text
GP28 -> 10 kOhm DNP -> base S8050 DNP
base -> 100 kOhm DNP -> GND
collecteur -> TP_EC25_RST_OD -> strap 0 Ohm DNP -> RESET_N
```

Cette option est exclusive avec NTC2_ADC.

---

# 9. Machine d'états Power

```text
BAT_DISARMED : BAT_ARM ouvert ; batterie isolée ; USB-C peut alimenter SYS
STORAGE      : MCU hard-off par STORAGE_SW
OFF          : MCU ON ; MAIN/MODEM/EC25_USB_VBUS/annexes OFF
BOOT_Q6A     : BQ valide -> MAIN ON -> PWR_ON_KEY -> attendre Q6A
RUN          : Q6A actif ; modem selon politique système
DEEP         : Android décide le suspend ; MAIN reste ON
SHUTDOWN_REQ : MCU demande arrêt complet Android
SHUTDOWN_READY : Android indique que MAIN peut être coupé
FAULT        : récupération graduelle ; hard cut en dernier recours
```

Cold boot :

```text
MCU boot
-> GPIO sûrs
-> BQ config + readback
-> MAIN_PWR ON
-> PWR_ON_KEY
-> attendre RUN
-> MODEM_PWR ON
-> attendre EC25_BAT_3V8 stable
-> EC25 PWRKEY
-> EC25_USB_VBUS ON
-> énumération USB
```

Shutdown propre :

```text
SHUTDOWN_REQ
-> Android arrête services/stockage/modem
-> SHUTDOWN_READY
-> EC25_USB_VBUS OFF
-> MODEM_PWR OFF
-> MAIN_PWR OFF
```

Reset MCU pendant RUN : RC MAIN/MODEM/USB_VBUS maintient les rails ~1 s ; GP10/GP29/GP6 sont réaffirmés en priorité.

---

# 10. SOC / batterie Android

Le MCU utilise VBAT / VSYS / IBAT / IBUS / TS / états charge/faults du BQ et peut exposer une batterie virtuelle via SBS. Ce n'est pas un fuel-gauge de précision. `MAX17048` non requis en baseline.

Le MCU doit disposer d'un seuil logiciel de batterie faible déclenchant un shutdown propre avant la coupure PCM JK50. Valeurs à définir au firmware après mesures de chute sous charge.

---

# 11. Connectique / pastilles THT

La batterie est la seule liaison interne avec connecteur PCB dédié. Les autres faisceaux utilisent des pastilles THT soudées.

```text
BAT_ARM       : BAT_RAW+ / BAT
BAT AUX       : ID1 / ID2 / NTC1 / NTC2 + TPs
Q6A PWR       : MAIN_PWR_OUT / GND
Q6A CTRL      : PWR_ON_KEY / SHUTDOWN_REQ / HEARTBEAT_STATE / SBS_SDA / SBS_SCL / Q6A_3V3_REF / GND
EC25 PWR      : EC25_BAT_3V8 / GND
EC25 USB PWR  : Q6A_EC25_VBUS_IN / EC25_USB_VBUS_OUT / GND
EC25 CTRL     : VIO / TXD / RXD / DTR / RI / PWRKEY / RESET_OD éventuel / GND
ANNEXE1       : ANNEXE1_OUT / GND
ANNEXE2       : ANNEXE2_OUT / GND
BUTTON        : POWER / GND
STORAGE       : 2 pads STORAGE_SW
RESERVE       : GP28 / GND
```

---

# 12. Testpoints obligatoires

```text
VBUS_USB_C / BQ_VBUS
BAT_RAW / BAT / SYS / GND
BAT_ID1 / BAT_ID2 / BAT_NTC1 / BAT_NTC2 / BQ_TS
MCU_VSYS / 3V3_MCU / TPS610995_EN
BQ_SDA / BQ_SCL / BQ_ILIM / BQ_PG
MAIN_PWR_GATE / MAIN_PWR_OUT
MODEM_PWR_GATE / MODEM_RAW
MODEM_REG_VIN / MODEM_REG_VOUT / TP_MODEM_PGOOD
EC25_BAT_3V8
EC25_USB_VBUS_IN / EC25_USB_VBUS_GATE / EC25_USB_VBUS_OUT
ANNEXE1_GATE / ANNEXE1_OUT
ANNEXE2_GATE / ANNEXE2_OUT
CC1 / CC2 / CC_SENSE
Q6A_3V3_REF / Q6A_PWRKEY / Q6A_SHUTDOWN_REQ / Q6A_HEARTBEAT_STATE
SBS_SDA / SBS_SCL
EC25_VIO / TXD / RXD / DTR / RI / PWRKEY
TP_GP28_3V3 / TP_NTC2_ADC / TP_EC25_RST_OD
```

---

# 13. PCB / mécanique / layout

```text
PCB : 4 couches
max strict : 70 x 25 mm
RP2040-Tiny : BOTTOM
```

```text
L1 : composants + USB2 extérieur + boucles puissance locales
L2 : GND continu
L3 : puissance + signaux lents
L4 : signaux + GND
```

Priorité placement :

```text
1. BQ25628E + ses boucles PMID/SW/SYS
2. J_BAT / BAT_ARM
3. RT6154A + L_MODEM + CIN/COUT, boucle très compacte
4. USB-C + protections
5. MAIN/MODEM high-side
6. switch EC25 USB_VBUS
7. annexes
8. logique / RP2040
```

Pour RT6154A : inductance collée à SW1/SW2, CIN/COUT au plus près, FB loin des nœuds SW, EP soudé sur plan PGND avec vias thermiques. La capacité **effective** des 47 uF à 3.825 V doit être vérifiée avec le derating DC réel.

---

# 14. BOM complète V0.9

## 14.1 Règles de lecture et prix

Cette BOM est la **baseline schéma/PCB**. `Qté` correspond à la population normale d'une carte ; les positions DNP sont indiquées séparément.

**Snapshot prix : 2026-10-05.** Les prix unitaires sont indicatifs en petite quantité et doivent être revalidés dans l'outil JLCPCB/LCSC au moment de la commande.

Pour JLCPCB **Economic PCBA**, le snapshot courant est :

```text
Basic / Promotional : pas de frais feeder Extended
Extended             : 3.00 USD par référence Extended unique et par commande
                       (pas 3.00 USD par exemplaire)
```

Le RP2040-Tiny est soudé manuellement après réception de la PCBA et n'est donc pas compté dans les frais Extended JLC de la carte assemblée.

## 14.2 Circuits intégrés, puissance, connecteurs et protections

| Bloc | Référence fabricant | LCSC/JLC | Qté | Classe JLC | Prix/u indicatif USD | Frais Extended/commande | Rôle précis |
|---|---|---:|---:|---|---:|---:|---|
| Charge/power-path | BQ25628ERYKR | C18221178 | 1 | Extended | ~2.53 | 3.00 | Chargeur JK50 1S, NVDC SYS, télémétrie ADC/I2C, TS, gestion courant d'entrée |
| Inductance BQ | XRIM252012S1R0MBCA 1 uH | C22471110 | 1 | Extended | ~0.04 | 3.00 | Inductance de conversion du BQ25628E |
| Alim MCU | TPS610995DRVR | C2071098 | 1 | Extended | ~1.75 | 3.00 | Boost ultra-basse consommation SYS -> MCU_VSYS 3.6 V, load disconnect |
| Inductance MCU | MAKK2016T2R2M 2.2 uH | C92923 | 1 | Extended | ~0.059 | 3.00 | Inductance du TPS610995 |
| Régulateur modem | RT6154AGQW | C250400 | 1 | Extended | ~1.01 | 3.00 | Buck-boost synchrone MODEM_RAW -> EC25_BAT_3V8 ~3.825 V |
| Inductance modem | FTC322520S2R2MBCA 2.2 uH | C5832397 | 1 | Extended | ~0.23 | 3.00 | Inductance RT6154A ; 5.4 A nominal, Isat ~6.5 A |
| Level shifter | SN74AVC4T245DR | C22495 | 1 | Extended | ~0.75 | 3.00 | Translation EC25 VIO ~1.8 V <-> MCU 3.3 V : TXD/RXD/DTR/RI, isolation partial-power-down |
| High-side principal | JMTQ55P02A | C2890429 | 2 | Extended | ~0.139 | 3.00 une fois | Q_MAIN pour Q6A et Q_MODEM pour MODEM_RAW ; faible Rds(on), hard cut + hold |
| USB-C | TYPE-C-31-M-12 | C165948 | 1 | Extended | ~0.186 | 3.00 | Connecteur extérieur charge 5 V + USB2 device ADB/EDL/MTP |
| ESD USB2 | SRV05-4 | C558418 | 1 | Extended | ~0.023 | 3.00 | Protection ESD D+/D- USB-C |
| TVS VBUS | SMF5.0A | C193402 | 1 | Extended | ~0.028 | 3.00 | Protection surtension/ESD du VBUS 5 V USB-C |
| Batterie | Molex 5050060812 | C779875 | 1 | Extended | ~0.14 | 3.00 | Connecteur JK50 : VBAT/GND/ID1/ID2/NTC1/NTC2 |
| Driver NMOS | AO3400A | C20917 | 5 | Basic | ~0.085 | 0 | Drivers gates MAIN, MODEM, ANNEXE1, ANNEXE2, EC25_USB_VBUS |
| High-side auxiliaire | AO3401A | C15127 | 3 | Basic | ~0.094 | 0 | P-MOS ANNEXE1, ANNEXE2 et switch USB_VBUS EC25 |
| Open collector | S8050 | C2146 | 2 | Basic | ~0.015 | 0 | Q6A PWR_ON_KEY + EC25 PWRKEY |
| Schottky USB | B5819W SL | C8598 | 1 | Basic | ~0.028 | 0 | USB-C VBUS -> Q6A OTG VBUS avec blocage du retour |
| MCU | Waveshare RP2040-Tiny | montage manuel | 1 | Hors PCBA | ~4.5 | 0 | Supervision always-on, power sequencing, boutons, BQ, modem, SBS |

Positions DNP supplémentaires utilisant des références existantes : `S8050 x1` pour RESET_N optionnel, aucun frais de référence supplémentaire.

## 14.3 Condensateurs

| Valeur / référence | LCSC | Qté montée | DNP | Classe | Prix/u indicatif | Rôle |
|---|---:|---:|---:|---|---:|---|
| 10 uF / 25 V | C15850 | 10 | 0 | Basic | ~0.011 | BQ PMID/SYS, TPS610995 entrée/sortie, RT6154 entrée |
| 47 uF / 10 V X5R 1206 | C96123 | 3 | 0 | Basic | ~0.07 | Réservoir sortie RT6154/EC25, 141 uF nominal avant derating |
| 100 nF | C1525 | 6 | 1 option NTC2 | Basic | ~0.001 | Découplages BQ/SN74/RT6154, CC_SENSE ; option filtre ADC NTC2 |
| 1 uF / 25 V | C52923 | 2 | 0 | Basic | ~0.012 | BQ VBUS et BAT |
| 1 uF RC hold | C15849 | 3 | 0 | Basic | ~0.018 | Hold MAIN, MODEM et USB_VBUS pendant reset MCU |
| 4.7 uF | C1779 | 1 | 0 | Basic | ~0.035 | REGN BQ25628E |
| 47 nF / 50 V | C1622 | 1 | 0 | Basic | ~0.007 | BTST BQ25628E |

Le compte final exact des 100 nF peut évoluer de +/-1 lors du placement des découplages sans changer de référence BOM.

## 14.4 Résistances et straps

| Valeur / référence | LCSC | Qté montée cible | DNP | Classe | Prix/u indicatif | Rôle |
|---|---:|---:|---:|---|---:|---|
| 5.6 kOhm 1 % | C23189 | 1 | 0 | Basic | <0.01 | RILIM BQ25628E |
| 5.1 kOhm | C25905 | 2 | 1 TS fallback | Basic | <0.01 | Rd CC1/CC2 USB-C ; position TS éventuelle non montée |
| 4.7 kOhm | C23162 | 0 | 2 | Basic | <0.01 | Pull-up SBS vers Q6A_3V3 si nécessaires |
| 10 kOhm | C25744 | ~11 | +2 DNP options | Basic | <0.01 | gates AO3400A, bases S8050, BQ I2C, bouton Power, SHUTDOWN série, options NTC/RESET |
| 100 kOhm | C25741 | ~12 | +2 DNP options | Basic | <0.01 | pulls gates/bases, SHUTDOWN, HEARTBEAT, Storage EN, options NTC/RESET |
| 470 kOhm | C23178 | 2 | 0 | Basic | <0.01 | CC1/CC2 -> CC_SENSE ADC |
| 1 MOhm | C22935 | 5 | 0 | Basic | <0.01 | hold gate pull-downs, HEARTBEAT pull-down, RT6154 R_TOP partielle |
| 330 kOhm | C23137 | 1 | 0 | Basic | <0.01 | RT6154 R_TOP avec 1 MOhm, total 1.33 MOhm |
| 200 kOhm | C25764 | 1 | 0 | Basic | <0.01 | RT6154 R_BOT ; fixe VOUT ~3.825 V |
| 30 kOhm | C22984 | 0 | 1 | Basic | <0.01 | TS fallback historique, DNP avant mesure JK50 |
| 0 Ohm | Basic à figer | 0 à 1 après choix | >=3 footprints | Basic | <0.01 | sélection NTC1/NTC2 -> BQ_TS, strap RESET_N optionnel |

Les quantités `~` sont celles attendues pour le schéma V0.9. Le générateur de BOM du schéma KiCad fera foi avant commande ; aucune nouvelle référence Extended ne doit être ajoutée silencieusement pour remplacer un passif Basic.

## 14.5 DNP / options explicites

| Option | Population baseline | Raison |
|---|---|---|
| SBS 2 x 4.7 kOhm vers Q6A_3V3 | DNP | Peupler uniquement si les pull-up Q6A ne sont pas utilisables |
| NTC1 -> BQ_TS strap | TBD après mesure | Sélection capteur thermique charge |
| NTC2 -> BQ_TS strap | TBD après mesure | Alternative à NTC1 ; jamais les deux simultanément |
| NTC2 -> GP28/ADC2 réseau | DNP | Deuxième température MCU après caractérisation |
| GP28 -> EC25 RESET_N open-drain | DNP | Recovery optionnel ; exclusif avec NTC2_ADC |
| PGOOD RT6154 pull-up vers MCU | DNP | TP disponible sans créer de back-power inutile |
| ID1/ID2 JK50 | TP / Hi-Z | Analyse/identification pack uniquement V1 |

## 14.6 Coût indicatif et frais Extended

Références Extended uniques PCBA V0.9 :

```text
1  BQ25628ERYKR
2  XRIM252012S1R0MBCA
3  TPS610995DRVR
4  MAKK2016T2R2M
5  RT6154AGQW
6  FTC322520S2R2MBCA
7  SN74AVC4T245DR
8  JMTQ55P02A
9  TYPE-C-31-M-12
10 SRV05-4
11 SMF5.0A
12 Molex 5050060812
```

Donc, snapshot Economic PCBA :

```text
12 références Extended x 3.00 USD = 36.00 USD de frais Extended par commande
```

Répartition indicative :

| Lot | Frais Extended totaux | Equivalent par carte |
|---:|---:|---:|
| 5 cartes | 36.00 USD | 7.20 USD/carte |
| 10 cartes | 36.00 USD | 3.60 USD/carte |
| 30 cartes | 36.00 USD | 1.20 USD/carte |

Coût composants indicatif :

```text
composants Extended montés, hors frais feeder : ~7.0 USD / carte
Basic + passifs : ordre de grandeur ~1.2 à 1.5 USD / carte
RP2040-Tiny manuel : ~4.5 USD / carte
électronique carte, hors PCB/assemblage/transport : ordre de grandeur ~13 USD / carte
```

Ces coûts sont des aides de décision, **pas des valeurs contractuelles**.

---

# 15. Fonctions explicitement absentes / interdites en V1

```text
USB-C host / DRP / contrôleur CC       absent
USB-PD / 9 V                           absent
GPIO séparé WAKE Q6A                   absent ; PWR_ON_KEY fait Power + wake
SLEEP_REQ forcé                        absent ; GP9 = SHUTDOWN_REQ
BQ_INT vers RP2040                     absent ; polling I2C
Volume +/- via RP2040                  absent ; Q6A directs
EC25 BAT direct depuis SYS             interdit en V0.9 ; RT6154A obligatoire
EC25 puissance principale via USB_VBUS interdite
USB EC25 D+/D- à travers PCB Power     absent ; données directes Q6A<->EC25
USB_VBUS EC25 non commutable           interdit ; GP6 + switch obligatoire
EC25 RESET dédié permanent             absent ; option DNP GP28
DISPLAY_PWR sur carte Power            absent
codec/jack/mux audio sur carte Power   absent
2e BMS/PCM batterie                    absent
fusible batterie obligatoire           absent baseline
Schottky/P-MOS anti-inversion BAT      absent baseline
```

---

# 16. Gates BLOQUANTES avant fabrication

## Batterie / BQ

```text
[ ] vraie JK50 reçue et mesurée
[ ] pinout/mating Molex vérifiés physiquement
[ ] NTC1/NTC2 caractérisés ; choix BQ_TS et réseau TS figés
[ ] BAT_ARM >=5 A et faible résistance
[ ] budget courant JK50 pire cas validé
[ ] BQ cold boot : watchdog/config/readback/fail-safe documentés
[ ] comportement BQ sans batterie / USB-only caractérisé
```

## RT6154 / modem power

```text
[ ] VOUT calculée/mesurée ~3.825 V avec tolérances
[ ] EC25_BAT_3V8 reste dans plage sûre sur toute plage SYS
[ ] stabilité RT6154 avec 2x10 uF entrée + 3x47 uF sortie réels
[ ] derating C96123 mesuré/estimé à 3.825 V
[ ] transitoires LTE sans reset modem/Q6A/BQ
[ ] thermique RT6154 + inductance au pire cas
[ ] MODEM_PWR OFF donne un vrai arrêt malgré USB/D+/D-
```

## Q6A / contrôle

```text
[ ] J19 mods et essais 4.2/3.8/3.4/3.0 V
[ ] PWR_ON_KEY cold boot/appui court/wake/long
[ ] SHUTDOWN_REQ GPIO58 + daemon Android
[ ] HEARTBEAT_STATE GPIO59
[ ] MAIN OFF + USB-C présent sans back-power problématique
[ ] reset MCU court : MAIN ne chute pas
[ ] SBS sans back-power
```

## EC25 / USB / level shift

```text
[ ] VIO/TXD/RI mesurés carrier réel
[ ] SN74AVC4T245 DIR/OE corrects et états OFF sûrs
[ ] switch Q6A->EC25 USB_VBUS fonctionne, OFF par défaut
[ ] MODEM_PWR OFF + USB_VBUS OFF = hard-off réel
[ ] D+/D- seuls ne back-powerent pas significativement le carrier
[ ] séquence MODEM_PWR -> 3V8 -> PWRKEY -> USB_VBUS validée
[ ] sleep EC25 : USB suspend puis, si utile, VBUS coupé avec MODEM_PWR ON
```

## PCB

```text
[ ] drivers annexes OFF garanti MCU reset/Hi-Z
[ ] TPS610995 shutdown/isolation/storage current
[ ] footprint RP2040-Tiny 1:1 + FPC accessible
[ ] layout BQ comparé référence TI
[ ] layout RT6154 comparé datasheet Richtek
[ ] USB-C D+/D- calculés stack-up JLC réel
[ ] ERC/DRC/BOM/PnP/polarités/Gerbers revus
[ ] PCB <=70 x 25 mm
```

---

# 17. Tests après fabrication

```text
BAT_ARM ouvert/fermé, USB présent/absent
USB-C -> BQ -> SYS sans batterie
batterie seule -> SYS
plug/unplug USB sans reboot
BQ : VREG/VSYSMIN/ICHG/TS/ADC/watchdog/readback
NTC1/NTC2 lecture et sécurité thermique
RP stable + Storage OFF courant résiduel
MAIN/MODEM/USB_VBUS OFF par défaut
reset MCU court : hold fonctionnel
RT6154 : VOUT, ripple, efficacité, température, transitoires
EC25 BAT ~3.825 V à batterie faible/nominale/haute + USB branché
Q6A PWR_ON_KEY court/wake/long
SHUTDOWN_REQ + SHUTDOWN_READY
USB-C ADB/EDL/MTP
EC25 USB énumération/coupure/réénumération
EC25 hard-off BAT + VBUS
EC25 D+/D- connectés modem OFF : back-power
EC25 UART/DTR/RI/PWRKEY
ANNEXE1/2 ON/OFF
consommations RUN / écran OFF / deep / OFF / storage-off
```

---

# 18. Sources primaires

- Texas Instruments — BQ25628E datasheet / NVDC power-path.
- Texas Instruments — TPS61099x documentation.
- Texas Instruments — SN74AVC4T245 documentation.
- Richtek — RT6154A/B datasheet, version courante.
- Quectel — EC25 Series Hardware Design + Reference Design.
- Radxa — Dragon Q6A V1.21 schematic + GPIO/USB documentation.
- Waveshare — RP2040-Tiny schematic/mécanique.
- Motorola repair schematics utilisant JK50.
- Molex — séries 505006/505004, drawing officiel.
- JLCPCB/LCSC — catalogue et règles Economic PCBA.

---

# 19. État de gel V0.9

```text
VALIDÉ / NORMATIF PCB-SCHÉMA
- JK50 + Molex 5050060812 + BAT_ARM
- BQ25628E power-path/charge, polling I2C
- MAIN_PWR JMTQ55P02A + AO3400A + hold
- MODEM_PWR JMTQ55P02A + AO3400A + hold
- MODEM_RAW -> RT6154AGQW -> EC25_BAT_3V8 ~3.825 V
- RT6154 : 2.2 uH C5832397, 2x10 uF entrée, 3x47 uF sortie
- feedback RT6154 : 1.33 MOhm / 200 kOhm
- PS/SYNC RT6154 LOW ; EN depuis MODEM_RAW
- EC25 USB VBUS commutable par GP6
- D+/D- EC25 directs Q6A <-> carrier
- ANNEXE1/2 AO3401A + AO3400A, OFF par défaut
- bouton Power court -> vrai PWR_ON_KEY
- GP9 = SHUTDOWN_REQ
- GP27 = HEARTBEAT_STATE
- GP28 pré-câblé NTC2_ADC DNP + RESET_N OD DNP, options exclusives
- NTC1/NTC2 sélectionnables vers BQ_TS par straps
- VOL+/VOL- directs Q6A
- BOM V0.9 ci-dessus = baseline d'implémentation

OUVERT / À CARACTÉRISER SANS MODIFIER L'ARCHITECTURE PCB
- choix NTC1 ou NTC2 pour BQ_TS et valeurs TS/JEITA
- population éventuelle du réseau NTC2_ADC
- tension VREG JK50 finale et paramètres BQ thermiques
- seuil batterie faible logiciel
- timings PWR_ON_KEY / SHUTDOWN / recovery
- codage temporel HEARTBEAT_STATE
- DIR/OE définitifs SN74AVC4T245 au schéma
- comportement back-power complet
- validation thermique et transitoire RT6154/EC25
```

Toute modification ultérieure d'un choix normatif doit être traitée comme une ECO ou une nouvelle révision d'architecture, jamais comme une correction silencieuse.
