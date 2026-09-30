# CDC — Carte Power / MCU V1 du Maker Phone Q6A

**Date :** 2026-09-30  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **CDC électrique + BOM V1 FIGÉS — prêt pour schéma/PCB**  
**Priorité :** ce document supersède les choix contradictoires des anciens documents Power/MCU.

> Fabrication imposée : **JLCPCB Economic PCBA**. Priorité composants : **Basic > Promotional > Extended**. Une référence Extended n'est gardée que lorsqu'une Basic/Promo correcte n'a pas été trouvée. Revalider uniquement stock/classification juste avant commande.

---

# 1. Architecture figée

```text
USB-C extérieur 5 V / USB2
        |
        +--> VBUS --> BQ25628E --> SYS -----------------------------+
        |                    |                                      |
        |                    +--> BAT --> batterie 1S + NTC         |
        |                                                           |
        +--> D+/D- -------------------------------> Q6A USB device  |
                                                                    |
SYS ----------------------------------------------------------------+
 |                                                                  |
 +--> TPS610995 --> 3V6 --> VSYS RP2040-Tiny                        |
 |                                                                  |
 +--> MAIN_PWR P-MOS -----------------------------> Q6A / J19        |
 +--> ANNEXE1_PWR P-MOS --------------------------> ANNEXE1 +/-      |
 +--> ANNEXE2_PWR P-MOS --------------------------> ANNEXE2 +/-      |

Q6A USB2 host
  VBUS / D+ / D- / GND ---------------------------> EC25 carrier

RP2040-Tiny
  <-> BQ25628E I2C + INT
  <-> Q6A GPIO58/GPIO59
  <-> EC25 TXD/RXD/DTR/RI/PWK/RST
  -> MAIN_PWR / ANNEXE1 / ANNEXE2
  <- bouton Power
  <- CC1/CC2 analogiques
```

Décisions générales :

```text
Batterie       : Li-ion/LiPo 1S ~5000 mAh, pack protégé
Chargeur       : BQ25628ERYKR / C18221178
USB-C          : 5 V uniquement, aucun PD V1
MCU            : Waveshare RP2040-Tiny soudé comme module
Alim MCU       : TPS610995DRVR -> 3.6 V -> VSYS du module
Q6A            : coupure physique MAIN_PWR
Modem          : alimenté par VBUS USB2 du Q6A ; aucun MODEM_PWR séparé
Écran          : aucun DISPLAY_PWR sur cette carte
Annexes        : deux sorties puissance commutées
PCB            : 4 couches
```

---

# 2. Batterie

[FIGÉ]

```text
Type           : Li-ion/LiPo 1S
Capacité cible : ~5000 mAh
Protection     : pack avec PCM/protection cellule obligatoire en V1 intégrée
NTC            : 10 kOhm type Semitec 103AT-2 ou strictement compatible
                 R25 = 10 kOhm ; B ~3435 K
```

Le NTC est en contact thermique avec la cellule et revient au connecteur batterie.

La sécurité de charge reste autonome : aucune protection critique ne dépend du RP2040 ou d'Android.

## Fusible

Pas de fusible assemblé par défaut. Prévoir sur le PCB :

```text
F_BAT : footprint optionnel
SJ_BAT: solder-jumper cuivre fermé par défaut
```

Si un fusible est monté plus tard, couper `SJ_BAT`. Le pack protégé reste obligatoire.

---

# 3. Chargeur / power-path BQ25628E

[FIGÉ]

```text
U_CHG       : BQ25628ERYKR
JLC         : C18221178
Boîtier     : WQFN-18 2.5 x 3 mm
Classe      : Extended / Economic
Batterie    : 1S
Charge max  : 2 A
Power-path  : NVDC intégré
ADC/I2C     : oui
```

Le shunt externe + ampli courant est supprimé. Le MCU exploite directement `VBAT`, `VSYS`, `VBUS`, `IBUS`, `IBAT`, TS, état de charge et défauts du BQ.

## 3.1 ILIM matériel au boot

[FIGÉ]

```text
RILIM = 5.1 kOhm / C25905 Basic
IILIM typique ~= 2500 / 5100 ~= 0.49 A
```

Le téléphone reste donc limité à environ **490 mA** tant que le firmware n'a pas pris la main.

`EN_EXTILIM` est un bit I2C, pas une broche. Séquence firmware obligatoire :

```text
1. conserver EN_EXTILIM actif au boot ;
2. déterminer le courant permis par la source ;
3. programmer IINDPM ;
4. seulement ensuite, si nécessaire, désactiver EN_EXTILIM.
```

## 3.2 Pins BQ

[FIGÉ]

```text
CE   -> GND
SDA  -> RP2040 + pull-up 10 kOhm vers 3V3
SCL  -> RP2040 + pull-up 10 kOhm vers 3V3
INT  -> RP2040 + pull-up 10 kOhm vers 3V3
PG   -> testpoint uniquement
STAT -> NC
QON  -> NC/testpoint, pull-up interne conservé
```

`CE=LOW` autorise matériellement la charge ; le MCU peut toujours agir par I2C via `EN_CHG`.

## 3.3 NTC / TS

[FIGÉ]

Topologie TI avec NTC 10 kOhm type 103AT :

```text
RT1 = 5.1 kOhm / C25905 Basic
RT2 = 30 kOhm  / C22984 Basic
NTC = 10 kOhm 103AT-2 compatible, hors PCB
```

Les valeurs Basic 5.1 kOhm / 30 kOhm remplacent 5.23 kOhm / 30.1 kOhm. L'écart est faible et sera validé thermiquement avant autorisation de charge 2 A.

## 3.4 Inductance et condensateurs

[FIGÉ]

```text
L_CHG    : XRIM252012S1R0MBCA / C22471110
           1 uH, 4 A rated, 5.6 A Isat, ~35 mOhm
           Extended / Economic

CVBUS    : 1 x 1 uF      / C52923 Basic
CVBUS_HF : 1 x 100 nF    / C1525 Basic
CPMID    : 2 x 10 uF     / C15850 Basic
CPMID_HF : 1 x 100 nF    / C1525 Basic
CSYS     : 3 x 10 uF     / C15850 Basic
CBAT     : 2 x 10 uF     / C15850 Basic
CREGN    : 1 x 4.7 uF    / C1779 Basic
CBTST    : 1 x 47 nF 50V / C1622 Basic, BTST-SW
```

Les condensateurs supplémentaires donnent de la marge après dérating DC des MLCC.

---

# 4. USB-C extérieur

[FIGÉ]

## 4.1 Connecteur

```text
TYPE-C-31-M-12 / C165948
USB-C femelle 16 pins / USB2
5 A / 20 V
Extended / Economic
```

```text
A6+B6 -> D+
A7+B7 -> D-
VBUS   -> TVS -> BQ VBUS
D+/D-  -> Q6A USB device
```

Pas de SuperSpeed, pas de SBU, pas de PD.

## 4.2 CC1 / CC2 + détection courant source

```text
CC1 -> 5.1 kOhm vers GND / C25905 Basic
CC2 -> 5.1 kOhm vers GND / C25905 Basic

CC1 -> 10 kOhm série / C25744 -> GP26 / ADC0
CC2 -> 10 kOhm série / C25744 -> GP27 / ADC1
```

Le RP2040 lit l'annonce Rp de la source USB-C. Si aucune annonce de courant fiable n'est reconnue, `ILIM` reste le garde-fou ~490 mA.

## 4.3 Protection USB

[FIGÉ]

```text
U_ESD : SRV05-4 / C558418
        4 canaux faible capacité
        D+, D-, CC1, CC2
        Extended / Economic

D_VBUS: SMF5.0A / C193402
        TVS 5 V, 200 W, SOD-123FL
        Extended / Economic
```

Pas de common-mode choke USB2 en V1. Routage différentiel direct, court, sur plan GND continu.

---

# 5. MCU RP2040-Tiny

[FIGÉ]

Le **Waveshare RP2040-Tiny** est soudé directement comme module sur la carte. Son footprint sera construit à partir des dimensions/pads officiels et vérifié sur le module réel avant fabrication.

## 5.1 Affectation GPIO

[FIGÉ]

```text
GP0   -> EC25 RXD via level-shifter       (UART TX MCU)
GP1   <- EC25 TXD via level-shifter       (UART RX MCU)
GP2   -> EC25 DTR via level-shifter
GP3   <- EC25 RI via level-shifter
GP4   <-> BQ SDA
GP5   ->  BQ SCL
GP6   <-  BQ INT
GP7   <-  PWR_BUTTON actif bas
GP8   ->  Q6A GPIO59 / MCU_WAKE
GP9   ->  Q6A GPIO58 / MCU_SLEEP_REQ
GP10  ->  MAIN_PWR_EN
GP11  ->  ANNEXE1_EN
GP12  ->  ANNEXE2_EN
GP13  ->  EC25 PWK driver
GP14  ->  EC25 RST driver
GP15  ->  AUX/debug
GP26  <-  CC1_SENSE / ADC0
GP27  <-  CC2_SENSE / ADC1
GP28  ->  AUX
GP29  ->  AUX
```

Bouton Power : contact externe NO entre `GP7` et GND, pull-up 10 kOhm / C25744 vers 3V3.

L'écran n'utilise aucun GPIO MCU dans cette V1 : `BL_EN`, `ENP`, `ENN` restent gérés côté Q6A/interposer écran.

---

# 6. Alimentation always-on du MCU

[FIGÉ]

Le MCU est alimenté depuis **SYS**, donc il reste disponible avec USB seul même sans batterie.

```text
SYS -> TPS610995DRVR -> 3.6 V -> VSYS RP2040-Tiny -> LDO 3.3 V module
```

```text
U_MCU_PWR : TPS610995DRVR / C2071098
L_MCU     : MAKK2016T2R2M / C92923
            2.2 uH, 1.5 A, Extended / Economic
CIN       : 1 x 10 uF / C15850 Basic
COUT      : 2 x 10 uF / C15850 Basic
EN        : relié à VIN/SYS, always-on
FB        : câblage variante fixe 3.6 V TI
```

Les inductances Basic disponibles ne fournissent pas une marge de courant comparable ; C92923 reste Extended justifiée.

---

# 7. MAIN_PWR / ANNEXE1 / ANNEXE2

[FIGÉ]

Même étage sur les trois branches :

```text
P-MOS : JMTQ55P02A / C2890429
        20 V, ~12 mOhm @ |VGS|=2.5 V
        Extended / Economic

Driver: S8050 / C2146 Basic
R_BASE: 10 kOhm  / C25744 Basic
R_BE  : 100 kOhm / C25741 Basic
R_GS  : 100 kOhm / C25741 Basic
```

Topologie :

```text
SYS -> source P-MOS
P-MOS drain -> charge
P-MOS gate -> 100 kOhm -> source
P-MOS gate -> collecteur S8050
S8050 émetteur -> GND
GPIO MCU -> 10 kOhm -> base
base -> 100 kOhm -> GND
```

Les rails sont OFF par défaut pendant reset/absence MCU.

`MAIN_PWR` est coupé normalement seulement après shutdown propre Android ; la coupure brutale est réservée à la récupération.

ANNEXE1/2 exposent uniquement `+` et `-`.

---

# 8. Q6A <-> MCU

[FIGÉ]

Le header Q6A et le RP2040 sont en logique 3.3 V : aucun level-shifter.

```text
Q6A pin 34 : GND
Q6A pin 36 : GPIO59 = MCU_WAKE
Q6A pin 37 : GPIO58 = MCU_SLEEP_REQ
```

Chaque ligne GPIO reçoit une résistance série **1 kOhm / C11702 Basic**. Cela limite un éventuel courant de back-power sans créer le diviseur potentiellement gênant qu'aurait une résistance série 10 kOhm face à un pull-up interne.

Le firmware privilégie LOW / Hi-Z quand la sémantique de la ligne le permet.

---

# 9. EC25 — USB Q6A

[FIGÉ]

```text
Q6A VBUS 5 V -> carrier VBUS
Q6A D+       -> carrier DP
Q6A D-       -> carrier DN
Q6A GND      -> carrier GND
```

```text
MODEM_PWR séparé : supprimé
VIN/BAT carrier   : non utilisés normalement
micro-USB carrier : non utilisé
```

Le carrier est câblé directement par pads/trous. D+/D- restent courts et torsadés si passage par fils.

La tenue du VBUS Q6A aux pics LTE reste un test de bring-up obligatoire.

---

# 10. EC25 <-> MCU

[FIGÉ]

**Aucun signal STATUS.** Le carrier réel n'expose pas STATUS. `NET` n'est pas assimilé à STATUS.

Signaux utilisés :

```text
TXD
RXD
DTR
RI
PWK
RST
VIO
GND
```

## 10.1 TXD / RXD / DTR / RI

```text
SN74AVC4T245DR / C22495
4 canaux, deux groupes de directions fixes
VCCA = VIO EC25 (~1.8 V attendu)
VCCB = 3V3 RP2040

EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
OE          : activé en permanence
DIR         : fixé matériellement, pas de GPIO MCU
Découplage  : 2 x 100 nF / C1525 Basic
```

La mesure réelle `VIO/TXD/RI` du carrier avant raccordement définitif reste obligatoire ; elle ne change pas la BOM.

## 10.2 PWK / RST

```text
2 x S8050 / C2146 Basic
2 x 10 kOhm / C25744 sur bases
2 x 100 kOhm / C25741 base->GND
```

Commande open-collector uniquement. Aucun pull-up 3.3 V ajouté côté collecteur.

## 10.3 Pins carrier non utilisées par le MCU

```text
NET                 : NC/testpad
PCM_IN/OUT/SYNC/CLK : pads extension audio future
SDA/SCL/AD0/AD1     : pads/testpoints domaine modem 1.8 V ; pas sur I2C MCU 3.3 V
VIN/BAT             : secours/test uniquement
```

---

# 11. Connecteurs

[FIGÉ]

Puissance : Molex Micro-Fit 3.0.

```text
J_BAT : 436500300 / C503478
        1x3 : BAT+ / GND / NTC

J_Q6A : 436500200 / C192562
        1x2 : MAIN_PWR+ / GND

J_A1  : 436500200 / C192562
        1x2 : ANNEXE1+ / GND

J_A2  : 436500200 / C192562
        1x2 : ANNEXE2+ / GND
```

Ces connecteurs THT sont montés manuellement après PCBA par défaut. Les boîtiers et cosses de câble sont hors BOM PCBA.

Debug/logique : footprint THT 1x10 pas 2.54 mm, montage manuel. Le modem utilise des pads/trous dédiés plutôt qu'un nouveau connecteur SMT.

---

# 12. PCB / routage

[FIGÉ]

PCB **4 couches** :

```text
L1 : composants + USB2/signaux critiques
L2 : GND continu
L3 : puissance + signaux lents
L4 : signaux + GND
```

Règles obligatoires : D+/D- différentiel sur plan continu ; ESD/TVS contre USB-C ; boucle SW/inductance/PMID très compacte ; découplages au plus près ; larges cuivres SYS/BAT/MAIN_PWR ; vias thermiques BQ ; éloigner SW de NTC/ADC/I2C/CC.

Le contour mécanique est déterminé lors du placement dans le châssis ; ce n'est plus une décision électrique.

---

# 13. BOM SMT gelée — 1 carte

| Qté | Fonction | Référence / valeur | JLC/LCSC | Classe |
|---:|---|---|---|---|
| 1 | Chargeur/power-path | BQ25628ERYKR | C18221178 | Extended / Economic |
| 1 | Inductance BQ | XRIM252012S1R0MBCA 1 uH | C22471110 | Extended / Economic |
| 1 | Boost MCU | TPS610995DRVR | C2071098 | Extended |
| 1 | Inductance boost MCU | MAKK2016T2R2M 2.2 uH | C92923 | Extended / Economic |
| 1 | Level-shifter EC25 | SN74AVC4T245DR | C22495 | Extended |
| 3 | P-MOS puissance | JMTQ55P02A | C2890429 | Extended / Economic |
| 5 | NPN drivers | S8050 J3Y | C2146 | **Basic** |
| 1 | USB-C | TYPE-C-31-M-12 | C165948 | Extended / Economic |
| 1 | ESD USB/CC 4 voies | SRV05-4 | C558418 | Extended / Economic |
| 1 | TVS VBUS | SMF5.0A | C193402 | Extended / Economic |
| 10 | MLCC | 10 uF / 25 V X5R 0805 | C15850 | **Basic** |
| 1 | MLCC | 1 uF | C52923 | **Basic** |
| 1 | MLCC | 4.7 uF | C1779 | **Basic** |
| 4 | MLCC | 100 nF | C1525 | **Basic** |
| 1 | MLCC bootstrap | 47 nF / 50 V X7R | C1622 | **Basic** |
| 4 | Résistance | 5.1 kOhm 1% | C25905 | **Basic** |
| 1 | Résistance NTC | 30 kOhm 1% | C22984 | **Basic** |
| 11 | Résistance | 10 kOhm 1% | C25744 | **Basic** |
| 8 | Résistance | 100 kOhm 1% | C25741 | **Basic** |
| 2 | Résistance Q6A | 1 kOhm 1% | C11702 | **Basic** |

Décompte 10 kOhm :

```text
BQ SDA/SCL       2
BQ INT           1
CC1/CC2 sense    2
PWR button       1
5 bases S8050    5
------------------
TOTAL           11
```

La BOM utilise donc massivement des références Basic pour les passifs. Les Extended restantes correspondent aux IC, inductances de puissance, MOS, USB-C et protections pour lesquels aucune Basic/Promo convaincante n'a été retenue.

---

# 14. Hors BOM SMT / montage manuel

```text
1 x Waveshare RP2040-Tiny
1 x NTC 10 kOhm type 103AT-2 compatible
1 x Molex 436500300 / C503478
3 x Molex 436500200 / C192562
1 x pin-header debug 1x10 P2.54 si monté
harness/fils EC25
boîtiers Micro-Fit + contacts à sertir
bouton Power NO du châssis
```

---

# 15. Testpoints obligatoires

```text
VBUS_USB_C
BAT
SYS
3V6_MCU
3V3_MCU
GND
BQ_SDA / BQ_SCL / BQ_INT / BQ_TS / BQ_ILIM
MAIN_PWR_GATE / MAIN_PWR_OUT
ANNEXE1_OUT / ANNEXE2_OUT
CC1 / CC2
EC25_VIO / TXD / RI / PWK / RST
Q6A_GPIO58 / Q6A_GPIO59
```

Il n'existe aucun testpoint `EC25_STATUS`.

---

# 16. Choix explicitement supprimés

```text
BQ25892                    -> remplacé par BQ25628E
shunt + ampli courant      -> supprimés
USB-C PD / 9 V             -> supprimé V1
MODEM_PWR                  -> supprimé
DISPLAY_PWR                -> supprimé
EC25 STATUS                -> supprimé
EC25 logique directe 3.3 V -> remplacée par SN74AVC4T245
CMC USB2                    -> non monté V1
fusible BAT assemblé       -> non monté ; footprint/jumper seulement
```

---

# 17. Ce qui reste à faire n'est plus une décision de BOM

```text
[ ] vérifier mécaniquement le footprint exact du RP2040-Tiny réel
[ ] mesurer EC25 VIO/TXD/RI avant connexion définitive
[ ] valider GPIO59 comme wake source STR sur le BSP réel
[ ] fixer contour et placement mécanique dans le châssis
[ ] revalider les stocks JLC juste avant commande
```

Une indisponibilité ponctuelle peut entraîner une substitution équivalente, mais ne doit pas rouvrir l'architecture.

---

# 18. Validation V1

```text
[ ] USB-C 5 V alimente SYS sans batterie
[ ] batterie seule alimente SYS
[ ] plug/unplug USB sans reboot involontaire
[ ] ILIM matériel ~0.5 A au boot
[ ] lecture CC1/CC2 correcte
[ ] IINDPM programmé avant désactivation éventuelle de EN_EXTILIM
[ ] charge 0.5 / 1 / 1.5 / 2 A validée thermiquement
[ ] NTC bloque/limite correctement hors plage
[ ] télémétrie BQ cohérente
[ ] RP2040 stable always-on
[ ] MAIN_PWR Q6A déterministe
[ ] ANNEXE1/2 déterministes
[ ] pas de back-power problématique Q6A OFF
[ ] GPIO58/59 suspend/wake validés
[ ] USB2 extérieur Q6A validé
[ ] EC25 USB direct validé
[ ] VBUS Q6A tient les pics LTE
[ ] UART EC25 via SN74AVC4T245 validé
[ ] PWK/RST/DTR/RI validés
[ ] consommation deep-off mesurée
[ ] Economic PCBA conservé
```
