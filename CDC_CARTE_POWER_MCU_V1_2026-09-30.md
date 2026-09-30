# CDC — Carte Power / MCU V1 du Maker Phone Q6A

**Date :** 2026-09-30  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **CDC électrique et BOM V1 gelés pour passage au schéma/PCB**  
**Priorité documentaire :** ce document supersède les choix contradictoires présents dans `PROJECT_STATE_POWER_USB_C_V1_2026-09-28.md`, `PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md` et les documents antérieurs concernant la carte Power/MCU.

> Règle fabrication : rester en **JLCPCB Economic PCBA**. Sélection composants : Basic en priorité, Promotional si pertinent, Extended uniquement lorsqu'aucune alternative Basic/Promo convenable n'existe. Les stocks/classes JLC doivent être revalidés juste avant commande, sans rouvrir l'architecture sauf indisponibilité réelle.

---

# 1. Architecture globale — FIGÉE

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
  <- CC1/CC2 analogiques pour lire l'annonce de courant USB-C
```

Décisions associées :

```text
Batterie                  : 1S Li-ion/LiPo ~5000 mAh, pack protégé
Chargeur                  : BQ25628ERYKR / C18221178
USB-C                     : 5 V uniquement, aucun PD V1
MCU                       : module Waveshare RP2040-Tiny soudé au PCB
Alim MCU                  : TPS610995DRVR -> VSYS du RP2040-Tiny
Q6A                       : coupure physique MAIN_PWR
Modem                     : alimenté par VBUS USB2 du Q6A, pas de MODEM_PWR séparé
Écran                     : pas de DISPLAY_PWR sur cette carte
Annexes                   : deux sorties puissance commutées indépendantes
PCB                       : 4 couches
```

---

# 2. Batterie — FIGÉ

```text
Chimie          : Li-ion / LiPo 1S
Capacité cible  : ~5000 mAh
Tension         : cellule 1S, typiquement ~3,0...4,2 V selon cellule retenue
Protection      : pack avec PCM/protection cellule obligatoire pour la V1 intégrée
Température     : NTC 10 kOhm type Semitec 103AT-2 ou strictement compatible
                 R25 = 10 kOhm, B ~3435 K
```

Le NTC est physiquement en contact avec la cellule et revient au connecteur batterie.

La sécurité de charge est autonome : elle ne doit pas dépendre d'Android ou du firmware RP2040.

## 2.1 Fusible batterie

Aucun fusible série n'est assemblé par défaut sur la V1 afin d'éviter résistance série et nouvelle référence Extended. Le PCB comporte :

```text
- un footprint optionnel F_BAT 1206/équivalent ;
- un solder-jumper cuivre SJ_BAT fermé par défaut ;
- sérigraphie indiquant de couper SJ_BAT avant montage d'un fusible futur.
```

Le pack protégé reste obligatoire. Le footprint de fusible est une option de révision, pas un composant de la BOM assemblée V1.

---

# 3. Chargeur / power-path — FIGÉ

```text
U_CHG       : BQ25628ERYKR
JLC         : C18221178
Boîtier     : WQFN-18 2.5 x 3 mm
Classe      : Extended, Economic PCBA
Batterie    : 1S
Charge max  : 2 A
Power-path  : NVDC intégré
ADC/I2C     : oui
BATFET      : intégré
```

La variante `E` est conservée. L'OTG boost supprimé sur cette variante n'est pas requis sur la V1.

La mesure externe shunt + ampli précédemment envisagée est supprimée. Le MCU utilise les mesures BQ : `VBAT`, `VSYS`, `VBUS`, `IBUS`, `IBAT`, TS, état charge et défauts.

La limite connue de la mesure IBAT pendant certains épisodes de battery supplement est acceptée pour la V1 et traitée logiciellement par recalage SOC.

---

# 4. BQ25628E — câblage définitif

## 4.1 Limite de courant d'entrée au boot

La limite matérielle ILIM est conservée comme garde-fou au reset :

```text
RILIM = 5.1 kOhm / C25905 Basic
KILIM typ ~2500 A.Ohm
IILIM typ ~2500 / 5100 = 0.49 A
```

Donc le téléphone démarre avec une limite d'entrée d'environ **490 mA** sans dépendre du firmware.

`EN_EXTILIM` est un bit I2C, pas une broche. Sa valeur POR active ILIM. Séquence firmware obligatoire :

```text
1. laisser EN_EXTILIM actif au boot ;
2. déterminer le courant autorisé par la source ;
3. programmer IINDPM ;
4. seulement ensuite, si utile, désactiver EN_EXTILIM.
```

Il est interdit au firmware de désactiver `EN_EXTILIM` avant d'avoir programmé `IINDPM`.

## 4.2 Pins logiques

```text
CE      : relié à GND -> charge matériellement autorisée ; EN_CHG I2C garde le contrôle logiciel
SDA     : vers RP2040, pull-up 10 kOhm vers 3V3
SCL     : vers RP2040, pull-up 10 kOhm vers 3V3
INT     : vers RP2040, pull-up 10 kOhm vers 3V3
PG      : testpoint uniquement, non utilisé par le MCU V1
STAT    : NC, laissé flottant
QON     : NC/testpoint ; pull-up interne du BQ utilisé
```

Le MCU reste always-on ; `QON`/ship-mode n'est pas utilisé comme commande normale de la V1.

## 4.3 Réseau NTC

Le réseau TI pour NTC 10 kOhm type 103AT est conservé dans sa topologie, mais rationalisé avec des valeurs Basic :

```text
NTC batterie : 10 kOhm 103AT-2 compatible, hors PCB / dans le pack
RT1          : 5.1 kOhm / C25905 Basic
RT2          : 30 kOhm / C22984 Basic
TS_BIAS      : pilote le réseau conformément au schéma d'application TI
TS           : point de mesure BQ
```

Les valeurs Basic 5.1 kOhm / 30 kOhm sont suffisamment proches des valeurs de référence 5.23 kOhm / 30.1 kOhm pour la V1. Les seuils thermiques réels seront vérifiés au bring-up avant autorisation de charge à 2 A.

## 4.4 Inductance et condensateurs BQ — FIGÉS

```text
L_CHG     : 1 uH XRIM252012S1R0MBCA / C22471110
            Extended / Economic
            4 A rated, 5.6 A saturation, DCR ~35 mOhm

CVBUS     : 1 x 1 uF  / C52923 Basic
CVBUS_HF  : 1 x 100 nF / C1525 Basic
CPMID     : 2 x 10 uF / C15850 Basic
CPMID_HF  : 1 x 100 nF / C1525 Basic
CSYS      : 3 x 10 uF / C15850 Basic
CBAT      : 2 x 10 uF / C15850 Basic
CREGN     : 1 x 4.7 uF / C1779 Basic
CBTST     : 1 x 47 nF 50 V X7R / C1622 Basic, entre BTST et SW
```

Les condensateurs 10 uF supplémentaires sont volontaires pour conserver de la marge après dérating DC des MLCC et respecter les minima effectifs du BQ.

---

# 5. USB-C extérieur — FIGÉ

## 5.1 Connecteur

```text
J_USB      : HRO TYPE-C-31-M-12
JLC        : C165948
Type       : USB-C femelle 16 pins, USB2
Rating     : 5 A / 20 V
Classe     : Extended / Economic
```

Pas de SuperSpeed. Pas de SBU. Pas de contrôleur PD.

```text
A6+B6 -> D+
A7+B7 -> D-
VBUS   -> protection VBUS -> BQ VBUS
D+/D-  -> Q6A USB device
```

## 5.2 CC1 / CC2

```text
R_CC1 = 5.1 kOhm vers GND / C25905 Basic
R_CC2 = 5.1 kOhm vers GND / C25905 Basic
```

En plus, le RP2040 mesure les deux lignes CC afin de connaître l'orientation et l'annonce de courant de la source Type-C :

```text
CC1 -> 10 kOhm série / C25744 -> RP2040 ADC0
CC2 -> 10 kOhm série / C25744 -> RP2040 ADC1
```

Le firmware ne relève `IINDPM` au-delà de la limite matérielle ~490 mA qu'après avoir identifié une annonce Type-C compatible. Sans information valide, il reste au courant par défaut conservateur.

## 5.3 ESD / surtension

```text
U_ESD_USB : SRV05-4 / C558418
            4 canaux, faible capacité
            protège D+, D-, CC1, CC2
            Extended / Economic

D_VBUS    : SMF5.0A / C193402
            TVS 5 V, SOD-123FL, 200 W
            Extended / Economic
```

Le TVS VBUS est placé au plus près du connecteur. Le BQ conserve en parallèle ses propres protections d'entrée.

**Pas de common-mode choke USB2 en V1.** Le routage D+/D- est direct, différentiel, court, au-dessus d'un plan GND continu. Cette décision économise une référence et évite d'ajouter une discontinuité inutile avant mesure EMI réelle.

---

# 6. MCU superviseur — FIGÉ

Le MCU est le **Waveshare RP2040-Tiny sous forme de module soudé directement au PCB**.

Le module est monté via son footprint castellated officiel/adapté mécaniquement au module réel. Il est soudé après assemblage JLC si nécessaire ; il n'impose pas de référence JLC.

Fonctions MCU :

```text
- bouton Power
- séquence ON/OFF Q6A
- STR wake/sleep Q6A
- MAIN_PWR / ANNEXE1 / ANNEXE2
- BQ25628E I2C + INT
- mesure CC1/CC2
- EC25 UART, DTR, RI, PWK, RST
- watchdog / récupération
```

L'écran n'utilise pas de `DISPLAY_PWR` sur cette carte. Les signaux BL_EN/ENP/ENN restent dans le domaine Q6A/interposer écran ; aucun GPIO MCU n'est consommé pour eux en V1.

## 6.1 Affectation GPIO RP2040 — FIGÉE

```text
GP0   -> EC25 RXD via level-shifter       (RP2040 TX)
GP1   <- EC25 TXD via level-shifter       (RP2040 RX)
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
GP15  ->  AUX_GPIO / debug
GP26  <-  CC1_SENSE ADC0
GP27  <-  CC2_SENSE ADC1
GP28  ->  AUX_GPIO
GP29  ->  AUX_GPIO
```

`GP8/GP9` sont câblés avec une résistance série 10 kOhm / C25744 vers le Q6A. Le firmware les utilise en LOW/Hi-Z lorsque possible afin de réduire le risque de back-power lorsque MAIN_PWR est coupé.

Le bouton Power externe est un contact NO entre `GP7` et GND avec pull-up 10 kOhm / C25744 vers 3V3. Le composant mécanique du bouton appartient au châssis, pas à la BOM SMT de cette carte.

---

# 7. Alimentation always-on RP2040-Tiny — FIGÉE

Le MCU est alimenté depuis **SYS**, pas directement depuis BAT : il doit rester disponible lorsque le téléphone est alimenté uniquement par USB.

```text
SYS
 -> TPS610995DRVR
 -> 3.6 V
 -> VSYS RP2040-Tiny
 -> LDO 3.3 V du module
```

```text
U_MCU_PWR : TPS610995DRVR / C2071098
            sortie fixe 3.6 V
L_MCU     : MAKK2016T2R2M / C92923
            2.2 uH, 1.5 A, Extended / Economic
CIN       : 1 x 10 uF / C15850 Basic
COUT      : 2 x 10 uF / C15850 Basic
EN        : relié au VIN/SYS -> always-on
FB        : câblé selon variante fixe 3.6 V TI
```

Aucune inductance Basic/Promo identifiée n'offre une marge de courant comparable dans ce format ; C92923 est donc conservée comme Extended justifiée.

---

# 8. MAIN_PWR et sorties ANNEXE — FIGÉS

Les trois rails utilisent le même étage pour réduire les références :

```text
Q_MAIN / Q_A1 / Q_A2 : JMTQ55P02A / C2890429
                        P-MOS 20 V
                        RDS(on) ~12 mOhm @ |VGS|=2.5 V
                        Extended / Economic

QDRV_MAIN/A1/A2      : S8050 / C2146 Basic
R_BASE               : 10 kOhm / C25744 Basic
R_BE                  : 100 kOhm / C25741 Basic
R_GS                  : 100 kOhm / C25741 Basic
```

Topologie :

```text
SYS -> source P-MOS
P-MOS drain -> charge
P-MOS gate -> 100 kOhm -> source
P-MOS gate -> collecteur S8050
S8050 émetteur -> GND
GPIO MCU -> 10 kOhm -> base S8050
base S8050 -> 100 kOhm -> GND
```

Ainsi les rails sont **OFF par défaut** si le MCU est en reset ou absent.

`MAIN_PWR` ne remplace pas un shutdown Android propre. Il sert au power-on depuis OFF et à la coupure physique après arrêt propre, avec coupure forcée seulement en récupération.

ANNEXE1 et ANNEXE2 n'exposent que `+` et `-`.

---

# 9. Q6A <-> MCU — FIGÉ

Le header GPIO Q6A utilisé ici est en logique 3.3 V ; le RP2040-Tiny est en 3.3 V.

```text
Q6A pin 34 : GND
Q6A pin 36 : GPIO59 = MCU_WAKE
Q6A pin 37 : GPIO58 = MCU_SLEEP_REQ
```

Aucun level-shifter n'est ajouté. Chaque ligne reçoit une résistance série 10 kOhm Basic pour limiter les courants de back-power accidentels pendant le deep-off.

La validation STR réelle de GPIO59 reste un test de bring-up, pas un choix de BOM.

---

# 10. Modem EC25 — USB Q6A — FIGÉ

Le carrier EC25 est alimenté et relié en données par l'USB2 host du Q6A :

```text
Q6A VBUS 5 V -> carrier VBUS
Q6A D+       -> carrier DP
Q6A D-       -> carrier DN
Q6A GND      -> carrier GND
```

```text
MODEM_PWR séparé : supprimé
VIN/BAT carrier   : non utilisés en fonctionnement normal
micro-USB carrier : non utilisé
```

Le raccordement est réalisé par pads/trous de câblage direct, avec paire D+/D- courte et torsadée si elle passe par fils. Aucun connecteur SMT supplémentaire n'est ajouté à cette liaison.

La tenue du VBUS Q6A pendant les pics LTE réels reste un test obligatoire avec SIM/data/appel.

---

# 11. Modem EC25 <-> MCU — FIGÉ

**Il n'y a PAS de signal STATUS dans la V1.** Le carrier réel utilisé n'expose pas STATUS ; il est supprimé du CDC, du schéma, des testpoints et des critères de validation. `NET` n'est pas considéré comme équivalent à STATUS.

Signaux MCU-modem retenus :

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

## 11.1 Translation 1.8 V / 3.3 V

```text
U_LS : SN74AVC4T245DR / C22495
       SOIC-16
       Extended

VCCA = VIO EC25 (~1.8 V attendu)
VCCB = 3V3 RP2040

A -> B : EC25 TXD, EC25 RI
B -> A : RP2040 TX -> EC25 RXD, RP2040 DTR -> EC25 DTR
OE     : activé en permanence
DIR    : fixé matériellement par groupe, aucune commande firmware

Découplage : 2 x 100 nF / C1525 Basic
```

La mesure physique de `VIO`, `TXD` idle et `RI` idle sur le carrier reste obligatoire avant première mise sous tension du lien ; elle valide le domaine ~1.8 V attendu sans modifier la BOM.

## 11.2 PWK / RST

PWK et RST sont commandés en open-collector :

```text
2 x S8050 / C2146 Basic
2 x R_BASE 10 kOhm / C25744 Basic
2 x R_BE   100 kOhm / C25741 Basic
```

Aucun pull-up 3.3 V n'est ajouté côté collecteur ; le transistor ne fait que tirer la ligne modem à GND.

## 11.3 Autres pins carrier

```text
NET                 : NC/testpad, pas vers MCU
PCM_IN/OUT/SYNC/CLK : pads d'extension audio futurs, non connectés au MCU V1
SDA/SCL/AD0/AD1     : pads/testpoints 1.8 V seulement ; pas sur le bus I2C 3.3 V MCU
VIN/BAT              : pads de secours/test uniquement, DNP en fonctionnement normal
```

---

# 12. Connectique — FIGÉE

Pour les chemins de puissance, la famille **Molex Micro-Fit 3.0** est retenue ; robuste et suffisamment dimensionnée.

```text
J_BAT : Molex 436500300 / C503478
        1x3, Micro-Fit 3.0, THT angle droit
        pins : BAT+ / GND / NTC

J_Q6A : Molex 436500200 / C192562
        1x2, Micro-Fit 3.0
        pins : MAIN_PWR+ / GND

J_A1  : Molex 436500200 / C192562
        pins : ANNEXE1+ / GND

J_A2  : Molex 436500200 / C192562
        pins : ANNEXE2+ / GND
```

Ces connecteurs sont THT/wave et peuvent être **montés manuellement après PCBA** pour éviter tout surcoût d'assemblage. Les boîtiers de câble et contacts à sertir sont hors BOM PCBA.

Connectique logique/debug : footprint pin-header THT 1x10 pas 2.54 mm, monté manuellement :

```text
GND
3V3
BQ_SDA
BQ_SCL
AUX_GP15
AUX_GP28
AUX_GP29
RUN/RESET si accessible
SWD/debug selon pads officiels RP2040-Tiny
GND
```

Le modem utilise des pads/trous de câblage dédiés plutôt qu'une nouvelle famille de connecteur SMT.

---

# 13. PCB / routage — FIGÉ

PCB **4 couches** :

```text
L1 : composants + signaux critiques/USB2
L2 : plan GND continu
L3 : distribution puissance + signaux lents
L4 : signaux + cuivre GND
```

Règles :

```text
- D+/D- USB2 en paire différentielle contrôlée, courte, sans coupure de plan sous la paire ;
- SRV05-4 et TVS au plus près du connecteur USB-C ;
- boucle BQ SW/inductance/PMID extrêmement compacte ;
- condensateurs BQ collés aux pins correspondantes ;
- larges polygones pour SYS, BAT, MAIN_PWR ;
- vias thermiques sous BQ25628E et composants power selon recommandations fabricant ;
- séparer le nœud SW des lignes ADC/I2C/CC/NTC ;
- testpoints accessibles au bring-up.
```

Le contour mécanique exact du PCB dépend du châssis et reste une donnée CAO, pas un choix électrique/BOM.

---

# 14. BOM SMT gelée — une carte

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
| 1 | ESD USB/CC 4 canaux | SRV05-4 | C558418 | Extended / Economic |
| 1 | TVS VBUS | SMF5.0A | C193402 | Extended / Economic |
| 10 | MLCC | 10 uF / 25 V X5R 0805 | C15850 | **Basic** |
| 1 | MLCC | 1 uF | C52923 | **Basic** |
| 1 | MLCC | 4.7 uF | C1779 | **Basic** |
| 4 | MLCC | 100 nF | C1525 | **Basic** |
| 1 | MLCC bootstrap | 47 nF / 50 V X7R | C1622 | **Basic** |
| 4 | Résistance | 5.1 kOhm 1% | C25905 | **Basic** |
| 1 | Résistance NTC | 30 kOhm 1% | C22984 | **Basic** |
| 14 | Résistance | 10 kOhm 1% | C25744 | **Basic** |
| 8 | Résistance | 100 kOhm 1% | C25741 | **Basic** |

Notes quantités résistances 10 kOhm :

```text
BQ SDA/SCL pull-up : 2
BQ INT pull-up     : 1
CC1/CC2 ADC série : 2
Q6A GPIO58/59 série:2
PWR_BUTTON pull-up : 1
5 bases S8050      : 5
----------------------
Total              : 13
```

**Réserver 14 positions BOM** : la quatorzième 10 kOhm est prévue comme spare configurable/test strap sur la V1. Elle peut être DNP si le schéma final ne l'utilise pas. Si JLC facture au placement et non à la référence, la laisser DNP ; la référence reste déjà présente dans la BOM.

> Remarque : la rationalisation privilégie les références Basic répétées. Les Extended conservées sont les fonctions pour lesquelles une alternative Basic/Promo adaptée n'a pas été trouvée sans dégrader courant, robustesse ou compatibilité électrique.

---

# 15. Composants hors BOM SMT / montage manuel

```text
1 x Waveshare RP2040-Tiny
1 x NTC 10 kOhm type 103AT-2 compatible sur batterie
1 x Molex 436500300 / C503478, batterie 3P
3 x Molex 436500200 / C192562, Q6A + Annexes 2P
1 x pin-header debug 1x10 2.54 mm, si souhaité
fils/harness EC25 USB + contrôle
boîtiers Micro-Fit et contacts à sertir côté câbles
bouton Power mécanique externe NO
```

Les connecteurs Molex peuvent aussi être assemblés en wave solder Economic, mais le choix V1 par défaut est montage manuel après PCBA.

---

# 16. Testpoints obligatoires

```text
VBUS_USB_C
BAT
SYS
3V6_MCU
3V3_MCU
GND
BQ_SDA
BQ_SCL
BQ_INT
BQ_TS
BQ_ILIM
MAIN_PWR_GATE
MAIN_PWR_OUT
ANNEXE1_OUT
ANNEXE2_OUT
CC1
CC2
EC25_VIO
EC25_TXD
EC25_RI
EC25_PWK
EC25_RST
Q6A_GPIO58
Q6A_GPIO59
```

**Aucun testpoint STATUS EC25.**

---

# 17. Décisions supprimées / explicitement non retenues

```text
BQ25892                    -> supprimé, BQ25628E retenu
shunt + ampli courant      -> supprimés
USB-C PD / 9 V             -> supprimé en V1
MODEM_PWR séparé           -> supprimé
DISPLAY_PWR                -> supprimé
EC25 STATUS                -> supprimé, non exposé par le carrier réel
EC25 logique directe 3.3 V -> supprimée ; level-shifter 1.8/3.3 V
CMC USB2                    -> non monté V1
fusible BAT assemblé       -> non monté ; footprint/jumper optionnel seulement
```

---

# 18. Ce qui reste à faire n'est plus une décision de BOM

Le schéma peut maintenant être dessiné sans nouveau choix de composant majeur. Restent uniquement des validations physiques/CAO :

```text
[ ] relever les dimensions exactes/footprint du RP2040-Tiny réel
[ ] mesurer EC25 VIO/TXD/RI avant connexion définitive
[ ] valider GPIO59 comme wake source STR sur le BSP réel
[ ] fixer le contour et les positions mécaniques du PCB dans le châssis
[ ] vérifier les stocks JLC juste avant commande
```

Une indisponibilité ponctuelle JLC peut justifier une substitution équivalente, mais ne doit pas rouvrir l'architecture.

---

# 19. Critères de validation V1

```text
[ ] USB-C 5 V alimente SYS sans batterie
[ ] batterie seule alimente SYS
[ ] plug/unplug USB sans reboot involontaire
[ ] ILIM matériel mesuré ~0.5 A au boot
[ ] lecture CC1/CC2 distingue correctement les annonces Type-C
[ ] firmware programme IINDPM avant toute désactivation de EN_EXTILIM
[ ] charge 0.5 / 1 / 1.5 / 2 A validée thermiquement
[ ] NTC bloque/limite correctement hors plage de température
[ ] télémétrie BQ cohérente via I2C
[ ] RP2040-Tiny reste stable sur le domaine always-on
[ ] MAIN_PWR démarre/coupe le Q6A de manière déterministe
[ ] ANNEXE1/ANNEXE2 commutent correctement
[ ] aucun back-power problématique vers Q6A quand MAIN_PWR=OFF
[ ] Q6A GPIO58/59 suspend/wake validés
[ ] USB2 extérieur Q6A fonctionne avec ESD monté
[ ] EC25 USB fonctionne via VBUS/DP/DN/GND directs
[ ] EC25 VBUS Q6A tient les pics LTE réels
[ ] UART EC25 fonctionne via SN74AVC4T245
[ ] PWK/RST/DTR/RI validés
[ ] consommation deep-off mesurée
[ ] aucune référence n'oblige à quitter Economic PCBA
```
