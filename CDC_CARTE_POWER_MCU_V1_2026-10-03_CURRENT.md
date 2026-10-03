# CDC — Carte Power / MCU V1 du Maker Phone Q6A — CURRENT

**Date :** 2026-10-04  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **CANDIDAT FINAL À VALIDATION UTILISATEUR — schéma/PCB V0.6**  
**Fabrication cible :** JLCPCB Economic PCBA  
**Priorité composants :** Basic > Promotional > Extended ; stock et classification à revalider avant commande.

Ce document est la référence électrique normative courante pour la carte Power/MCU. Il supersède les anciens CDC Power/MCU lorsqu'ils sont contradictoires.

Décisions ajoutées lors de la passe finale du 04/10 :

```text
- aucun connecteur de faisceau sur la carte Power : pastilles THT pour fils soudés ;
- BQ25628E INT n'est plus relié au MCU ; surveillance par polling I2C ;
- GP6 devient GPIO de réserve ;
- GP28 reste GPIO de réserve + option EC25 RESET_N open-drain DNP ;
- Volume+ / Volume- ne passent pas par le MCU ; ils vont directement au Q6A ;
- VOL+ = Q6A J20 pin 29 / GPIO31 ;
- VOL- = Q6A J20 pin 32 / GPIO30 ;
- ANNEXE1 = alimentation commutée de l'ampli haut-parleur ;
- ANNEXE2 = rail SYS commuté générique pour caméras / flash / outils / annexes futures.
```

---

# 0. Architecture finale retenue

```text
USB-C extérieur 5 V / USB2 DEVICE uniquement
        |
        +--> VBUS --> TVS --> BQ25628E --> SYS -----------------------------+
        |                         |                                          |
        |                         +--> BAT --> batterie 1S protégée + NTC    |
        |                                                                    |
        +--> D+/D- -----------------------------> Q6A USB OTG DEVICE         |
        +--> VBUS -- Schottky ------------------> Q6A OTG VBUS               |
                                                                             |
SYS -------------------------------------------------------------------------+
 |                                                                            |
 +--> TPS610995 --> MCU_VSYS --> VSYS RP2040-Tiny                             |
 |       ^                                                                    |
 |       +-- EN <- STORAGE_SW <- SYS ; EN pulldown 100 kOhm                  |
 |                                                                            |
 +--> MAIN_PWR : JMTQ55P02A --> Q6A J19                                      |
 |       commande AO3400A + maintien RC ~1 s                                 |
 |                                                                            |
 +--> MODEM_PWR : JMTQ55P02A --> EC25 carrier BAT                            |
 |       commande AO3400A + maintien RC ~1 s                                 |
 |                                                                            |
 +--> ANNEXE1 : AO3401A, ampli haut-parleur, OFF par défaut                   |
 +--> ANNEXE2 : AO3401A, annexes téléphone, OFF par défaut                    |

Q6A USB2 HOST #1 ---- VBUS/D+/D-/GND ----> EC25 VBUS/DP/DN/GND
Q6A USB2 HOST #2 -------------------------> caméra USB future
Q6A USB2 HOST #3 -------------------------> réserve / futur USB-A amovible

RP2040-Tiny
  <-> BQ25628E I2C ; états/faults lus par polling
  <-> Q6A SBS/I2C
   -> Q6A PWR_ON_KEY via S8050
   -> Q6A SLEEP_REQ
   <- Q6A HEARTBEAT
  <-> EC25 UART/DTR/RI via SN74AVC4T245
   -> EC25 PWRKEY via S8050
   -> MAIN_PWR / MODEM_PWR / ANNEXE1 / ANNEXE2
   <- bouton Power utilisateur
   <- CC_SENSE analogique fusionné

Boutons Volume
  VOL+ -> Q6A GPIO31 direct
  VOL- -> Q6A GPIO30 direct
  aucune liaison vers RP2040
```

La carte Power ne transporte ni le DSI écran, ni le tactile, ni l'USB Q6A <-> EC25. L'écran conserve son interposer dédié. La liaison USB EC25 est un faisceau court direct.

---

# 1. Batterie et domaine SYS

## 1.1 Batterie

```text
Type            : Li-ion/LiPo 1S classique
Tension max     : 4.20 V ; cellule HV 4.35/4.40 V interdite
Capacité cible  : ~5000 mAh
Protection      : pack protégé / PCM obligatoire
NTC             : 10 kOhm type Semitec 103AT-2 ou strictement compatible
Courant cible   : pack/PCM/fils capables d'encaisser Q6A + LTE + annexes
```

Objectif de sélection pack : **>=5 A continu avec marge de pic**, à confirmer sur la cellule et son PCM réels.

Toutes les liaisons batterie sont faites par grosses pastilles traversantes pour fils soudés :

```text
BAT+
BAT-
NTC
```

Aucun connecteur Molex/JST n'est imposé par la PCB.

## 1.2 Fusible batterie

```text
F_BAT  : footprint optionnel DNP
SJ_BAT : jumper cuivre fermé par défaut
```

Le pack protégé reste obligatoire.

---

# 2. BQ25628E — chargeur / power-path

## 2.1 Référence

```text
U_CHG       : BQ25628ERYKR
JLC/LCSC    : C18221178
Boîtier     : WQFN-18 2.5 x 3 mm
Batterie    : 1S
Charge max  : 2 A
Power-path  : NVDC
ADC/I2C     : oui
```

Le BQ fournit VBUS/VBAT/VSYS/IBUS/IBAT/TS et les états charge/fault. Il n'est pas considéré comme fuel-gauge de précision.

## 2.2 Pins et politique INT

```text
CE   -> GND
SDA  -> GP4 + pull-up 10 kOhm vers 3V3_MCU
SCL  -> GP5 + pull-up 10 kOhm vers 3V3_MCU
INT  -> non relié au MCU ; NC, TP optionnel seulement si placement gratuit
PG   -> testpoint
STAT -> NC
QON  -> testpoint / pull-up interne conservé
```

**Décision finale : GP6 n'est plus consommé par BQ_INT.**

Le MCU surveille le BQ par polling I2C. Les protections et la charge sûre ne doivent pas dépendre d'une interruption ou du firmware MCU.

Politique firmware initiale :

```text
RUN / charge active : polling typique ~1 s
veille MCU          : cadence ralentie selon besoin
réveil / événement  : lecture immédiate de tous les états utiles
```

La cadence exacte reste firmware ; elle ne modifie pas le PCB.

## 2.3 ILIM et configuration de départ

```text
RILIM = 5.6 kOhm / C23189 Basic / 1 %
VREG nominal        : 4.15 V
VSYSMIN             : 3.84 V
source inconnue     : IINDPM conservateur ~0.44 A
Rp 1.5 A reconnu    : ~1.35 A max de départ
Rp 3.0 A reconnu    : <= capacité réelle entrée/BQ
ICHG                : <=2 A et réduit selon thermique / charge système
```

`EN_EXTILIM` reste actif tant que l'annonce Type-C n'est pas reconnue de façon fiable.

## 2.4 MODEM_PWR et plage SYS

Politique de départ :

```text
MODEM_PWR autorisé si environ 3.35 V <= VSYS <= 4.25 V
```

Gate absolue :

```text
EC25 carrier BAT doit rester < 4.30 V dans tous les états,
y compris batterie pleine, USB branché et transitoires.
```

## 2.5 NTC / TS

```text
RT1 = 5.1 kOhm / C25905 Basic
RT2 = 30 kOhm  / C22984 Basic
NTC = 10 kOhm 103AT-2 compatible, hors PCB
```

Validation thermique obligatoire avant charge 2 A.

## 2.6 Passifs principaux

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

## 2.7 Layout BQ

```text
- PMID caps collés au pin PMID/GND ;
- boucle PMID -> switch -> inductance -> SYS -> caps -> GND minimale ;
- SYS caps collés au pin SYS ;
- CVBUS/CBAT/REGN au plus près ;
- SW le plus petit possible ;
- vias GND/thermiques sous et autour du BQ ;
- SW éloigné de TS, I2C, CC et ADC ;
- BAT/SYS/MAIN/MODEM très larges ;
- vias multiples sur changements de couche puissance.
```

---

# 3. USB-C extérieur — charge + device uniquement

## 3.1 Fonction figée

```text
Charge 5 V
ADB
EDL
MTP / USB device
```

**Aucun host USB-C, aucun DRP, aucun contrôleur CC, aucun PD en V1.**

## 3.2 Connecteur et données

```text
TYPE-C-31-M-12 / C165948
A6+B6 -> D+ -> ESD -> Q6A OTG D+
A7+B7 -> D- -> ESD -> Q6A OTG D-
VBUS  -> TVS -> BQ VBUS
```

D+/D- : vraie paire USB2 ~90 Ohm différentiel sur stack-up JLC réel, plan GND continu, aucun stub, minimum de vias.

## 3.3 VBUS vers Q6A

```text
USB-C VBUS -> anode B5819W SL / C8598
cathode    -> Q6A OTG VBUS
```

Gate : USB-C branché + MAIN_PWR OFF ne doit pas back-powerer significativement le Q6A par VBUS ou D+/D-.

## 3.4 CC1/CC2 et détection courant

```text
CC1 -> 5.1 kOhm -> GND
CC2 -> 5.1 kOhm -> GND

CC1 -- 470 kOhm --+
                   +--> CC_SENSE -> GP26 / ADC0
CC2 -- 470 kOhm --+
                   |
                 100 nF
                   |
                  GND
```

Tout niveau ambigu est traité comme source Default/conservatrice.

## 3.5 Protection

```text
U_ESD  : SRV05-4 / C558418 — D+, D-, CC1, CC2
D_VBUS : SMF5.0A / C193402 — VBUS
```

Pas de common-mode choke USB2 en V1.

---

# 4. RP2040-Tiny / always-on / Storage OFF

## 4.1 Module

Waveshare RP2040-Tiny soudé manuellement face BOTTOM, sans headers.

```text
- footprint contrôlé sur module réel ;
- impression 1:1 obligatoire ;
- FPC accessible après assemblage ;
- keepout mécanique sous module ;
- aucun via/testpoint pouvant toucher le dessous du module.
```

Le FPC du module reste le chemin USB/BOOTSEL/RUN de récupération.

## 4.2 Alimentation

```text
SYS -> TPS610995DRVR / C2071098 -> MCU_VSYS -> VSYS RP2040-Tiny
L_MCU : MAKK2016T2R2M / C92923 / 2.2 uH
CIN   : 10 uF
COUT  : 2 x 10 uF
```

## 4.3 Storage switch

```text
SYS -> STORAGE_SW -> EN TPS610995
EN -> 100 kOhm -> GND
```

```text
fermé  : MCU alimenté
ouvert : hard-off MCU / stockage
```

Storage OFF ne remplace jamais le shutdown normal.

## 4.4 Pinout RP2040-Tiny final

```text
GP0   -> EC25 RXD via SN74AVC4T245       UART TX MCU
GP1   <- EC25 TXD via SN74AVC4T245       UART RX MCU
GP2   -> EC25 DTR via SN74AVC4T245
GP3   <- EC25 RI via SN74AVC4T245
GP4   <-> BQ SDA
GP5   ->  BQ SCL
GP6   <-> RESERVE / pastille THT + testpoint
GP7   <-  PWR_BUTTON utilisateur, actif bas
GP8   ->  Q6A PWR_ON_KEY via S8050
GP9   ->  Q6A SLEEP_REQ, actif bas / Hi-Z au repos
GP10  ->  MAIN_PWR_EN avec RC hold
GP11  ->  ANNEXE1_EN / ampli haut-parleur
GP12  ->  ANNEXE2_EN / annexes téléphone
GP13  ->  EC25 PWRKEY via S8050
GP14  <-> Q6A SBS_SDA
GP15  <-> Q6A SBS_SCL
GP16  : WS2812 du module ; aucune fonction système obligatoire
GP26  <-  USB_CC_SENSE / ADC0
GP27  <-  Q6A HEARTBEAT / GPIO59
GP28  <-> RESERVE / TP 3.3 V / option EC25 RESET_N OD DNP
GP29  ->  MODEM_PWR_EN avec RC hold
```

**Deux réserves pratiques sont donc conservées : GP6 et GP28.**

GP6 doit être exposé simplement, sans fonction imposée. GP28 conserve en plus l'étage RESET_N optionnel DNP décrit plus bas.

---

# 5. MAIN_PWR et MODEM_PWR — OFF par défaut + maintien reset MCU

## 5.1 Puissance

```text
Q_MAIN_P  : JMTQ55P02A / C2890429
Q_MODEM_P : JMTQ55P02A / C2890429
```

Les deux rails sont OFF si le MCU est absent ou durablement arrêté.

## 5.2 Driver commun

Par rail :

```text
P-MOS source -> SYS
P-MOS drain  -> charge
P-MOS gate   -> 100 kOhm -> SYS
P-MOS gate   -> drain AO3400A
AO3400A source -> GND
GPIO MCU -> 10 kOhm -> gate AO3400A
AO gate  -> 1 MOhm -> GND
AO gate  -> 1 uF -> GND
```

Composants :

```text
AO3400A : C20917
10 kOhm : C25744
100 kOhm: C25741
1 MOhm  : C22935
1 uF    : C15849
```

Cible hold mesurée : ~0.7–1.5 s.

```text
GPIO HIGH -> ON
GPIO LOW  -> OFF rapide
GPIO Hi-Z -> maintien temporaire par RC
```

Au boot MCU, GP10 et GP29 sont traités en priorité absolue pour réaffirmer HIGH avant expiration du hold lorsqu'un rail était déjà actif.

---

# 6. ANNEXE1 / ANNEXE2

Branches simples, **OFF par défaut**, sans RC de maintien.

```text
P-MOS : AO3401A / C15127 Basic
cible : ~1 A continu max par annexe, sous réserve thermique/layout
```

## 6.1 ANNEXE1

Fonction figée : **alimentation commutée de l'ampli haut-parleur**.

```text
GP11 -> ANNEXE1_EN
SYS -> ANNEXE1_OUT -> ampli
```

Le signal audio lui-même reste géré par le Q6A / chaîne audio dédiée, hors fonction Power de cette carte.

## 6.2 ANNEXE2

Fonction figée : **rail générique de mise sous tension des annexes téléphone**.

Usages prévus :

```text
caméras
flash / éclairage
capteurs ou outils téléphone
petits modules futurs
```

```text
GP12 -> ANNEXE2_EN
SYS -> ANNEXE2_OUT
```

Important : ANNEXE2 est du **SYS commuté non régulé**. Toute annexe nécessitant 5 V, 3.3 V fixe ou une autre tension doit posséder son propre convertisseur/régulateur aval.

---

# 7. Q6A <-> carte Power / MCU

## 7.1 Liaisons finales

```text
Q6A J20 pin 36 / GPIO59 -> HEARTBEAT -> GP27
Q6A J20 pin 37 / GPIO58 <- SLEEP_REQ <- GP9
Q6A J20 pin 3  / GPIO24 / I2C6_SDA <-> GP14 SBS_SDA
Q6A J20 pin 5  / GPIO25 / I2C6_SCL <-> GP15 SBS_SCL
Q6A J20 pin 1 ou 17 / 3V3 -> Q6A_3V3_REF
Q6A GND -> GND
Q6A PWR_ON_KEY <- S8050 <- GP8
MAIN_PWR_OUT -> Q6A J19+
GND          -> Q6A J19-
```

Toutes ces liaisons de faisceau sont sur **pastilles THT pour fils soudés**, pas sur connecteurs PCB.

## 7.2 Configuration batterie Q6A J19

Configuration cible :

```text
R7    -> retiré / DNP
R24   -> 2 mOhm 1 %
R190  -> 100 kOhm vers GND
R191  -> 10 kOhm vers GND
FB4   -> DNP
R185..R189 -> DNP si inspection confirme
```

J19 devient l'entrée 1S principale du Q6A.

## 7.3 PWR_ON_KEY

```text
GP8 -> 10 kOhm -> base S8050
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Fonctions : cold boot, wake deep, événement Power Android, appui long recovery si nécessaire.

Pulse initial à caractériser autour de 100–300 ms.

## 7.4 SLEEP_REQ

```text
Q6A_3V3 -> 100 kOhm -> SLEEP_REQ
SLEEP_REQ -> 10 kOhm série -> GP9
```

MCU : actif = sortie LOW ; repos = input/Hi-Z.

SLEEP_REQ déclenche la procédure complète de suspend : EC25, USB host, OTG externe, wake sources, puis `mem_sleep=deep`.

## 7.5 HEARTBEAT

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Le heartbeat est un statut logiciel. Son absence seule ne coupe jamais MAIN_PWR.

## 7.6 SBS / batterie virtuelle Android

```text
GP14 <-> Q6A I2C6_SDA
GP15 <-> Q6A I2C6_SCL
```

Prévoir :

```text
2 x 4.7 kOhm vers Q6A_3V3, DNP par défaut
```

Ne jamais tirer SBS vers 3V3_MCU. Mesurer d'abord les pull-up déjà présents sur le Q6A.

---

# 8. Boutons utilisateur

## 8.1 Power

Le bouton Power physique reste géré uniquement par le MCU :

```text
GP7 <- bouton NO vers GND
pull-up 10 kOhm vers 3V3_MCU
```

Le firmware décide cold boot / SLEEP_REQ / wake PWR_ON_KEY / recovery.

## 8.2 Volume + / Volume -

Décision finale : **les boutons Volume ne passent pas par le MCU.**

```text
VOL+ : Q6A J20 pin 29 / GPIO31
VOL- : Q6A J20 pin 32 / GPIO30
```

Pour chaque bouton :

```text
Q6A_3V3
  |
 10 kOhm
  |
GPIO ---- bouton NO ---- GND
```

Entrées actives bas, déclarées côté Linux comme `gpio-keys` / événements volume natifs.

Aucune liaison VOL+/VOL- vers GP6, GP28 ou un ADC RP2040.

Ces boutons sont hors logique Power board ; ils sont câblés directement vers le Q6A par fils soudés / header adapté à l'intégration mécanique finale.

---

# 9. EC25 — alimentation, USB, supervision

## 9.1 Alimentation principale

```text
SYS -> MODEM_PWR -> EC25 carrier BAT
GND -> EC25 carrier GND
```

BAT porte l'énergie LTE. USB_VBUS n'est pas l'alimentation principale du modem.

MODEM_PWR et GND utilisent de grosses pastilles THT pour fils soudés.

## 9.2 USB direct Q6A -> EC25

```text
Q6A USB2 host VBUS -> EC25 VBUS
Q6A USB2 host D+   -> EC25 DP
Q6A USB2 host D-   -> EC25 DN
Q6A GND            -> EC25 GND
```

Cette liaison ne traverse pas la carte Power.

Pas de switch VBUS EC25 en V1.

## 9.3 UART / DTR / RI

```text
SN74AVC4T245DR / C22495
VCCA = EC25 VIO (~1.8 V)
VCCB = 3V3_MCU

EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

```text
VIO -> 100 kOhm -> GND
EC25 DTR -> 100 kOhm -> VIO
EC25 RXD -> 100 kOhm -> VIO
100 nF sur VCCA
100 nF sur VCCB
```

VIO/TXD/RI doivent être mesurés sur le carrier réel avant connexion définitive.

## 9.4 PWRKEY

```text
GP13 -> 10 kOhm -> base S8050
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> EC25 PWRKEY
```

Recovery : AT propre -> PWRKEY -> MODEM_PWR power-cycle.

## 9.5 RESET_N / réserve GP28

GP28 reste une vraie réserve 3.3 V :

```text
GP28 -> TP/PASTILLE_GP28_3V3
```

Étage DNP optionnel :

```text
GP28 -> 10 kOhm DNP -> base S8050 DNP
base -> 100 kOhm DNP -> GND
émetteur -> GND
collecteur -> TP_EC25_RST_OD -> strap DNP -> EC25 RESET_N
```

Ce chemin est open-drain ; ce n'est pas une sortie push-pull 1.8 V.

---

# 10. Machine d'états Power

```text
STORAGE : MCU/Q6A/EC25 OFF
OFF     : MCU ON ; MAIN/MODEM OFF
BOOT    : MAIN ON -> délai PMIC -> PWR_ON_KEY -> heartbeat -> MODEM si SYS sûr
RUN     : MAIN ON ; fonctionnement normal
DEEP    : MAIN ON ; MODEM éventuellement ON ; heartbeat peut s'arrêter volontairement
SHUTDOWN: Android + modem arrêtés proprement -> MODEM OFF -> MAIN OFF
FAULT   : récupération graduelle ; hard cut seulement en dernier recours
```

## Reset MCU pendant RUN

```text
RC MAIN/MODEM maintient ~1 s
-> RP2040 reboot
-> GP10/GP29 réaffirmés immédiatement
-> pas de reset Q6A/modem attendu
```

## Suspend normal

```text
Power utilisateur -> MCU
-> SLEEP_REQ
-> EC25 QSCLK/DTR + USB host suspend
-> USB OTG externe -> none si requis
-> gestion Wi-Fi/wake sources
-> mem_sleep=deep
```

Réveil : PWR_ON_KEY.

## Shutdown normal

```text
1. shutdown Android propre
2. arrêt EC25 propre
3. MODEM_PWR OFF
4. MAIN_PWR OFF
5. MCU reste ON tant que STORAGE_SW est fermé
```

---

# 11. SOC / batterie Android

Le MCU utilise :

```text
VBAT / VSYS / IBAT / IBUS / TS
état charge/faults
courbe OCV/SOC
intégration logicielle lorsque pertinente
```

Exposition via SBS possible. Ce n'est pas un fuel-gauge de précision.

`MAX17048` n'est pas requis. Un footprint DNP éventuel ne doit jamais être une dépendance V1.

---

# 12. Faisceaux / pastilles THT

## 12.1 Règle générale

**Aucun connecteur de faisceau n'est retenu sur cette carte.**

Les liaisons externes sont des pastilles traversantes percées, assez grandes pour soudure manuelle et reprise mécanique, clairement sérigraphiées.

Groupes minimum :

```text
BAT      : BAT+ / BAT- / NTC
Q6A PWR  : MAIN_PWR_OUT / GND
Q6A CTRL : PWR_ON_KEY / SLEEP_REQ / HEARTBEAT / SBS_SDA / SBS_SCL / Q6A_3V3_REF / GND
EC25 PWR : MODEM_PWR_OUT / GND
EC25 CTRL: VIO / TXD / RXD / DTR / RI / PWRKEY / éventuellement RESET_OD / GND
ANNEXE1  : ANNEXE1_OUT / GND
ANNEXE2  : ANNEXE2_OUT / GND
BUTTON   : POWER / GND
STORAGE  : 2 pads pour interrupteur STORAGE_SW
RESERVE  : GP6 / GND ; GP28 / GND
```

Les sorties de puissance doivent utiliser des trous/pads plus gros que les signaux logiques.

Prévoir soulagement mécanique / possibilité d'attacher les faisceaux hors pads si la mécanique finale le permet ; les pads ne doivent pas reprendre seuls les efforts d'arrachement.

---

# 13. Testpoints obligatoires

```text
VBUS_USB_C / BQ_VBUS / BAT / SYS
MCU_VSYS / 3V3_MCU / TPS610995_EN / GND
BQ_SDA / BQ_SCL / BQ_TS / BQ_ILIM / BQ_PG
MAIN_PWR_GATE / MAIN_PWR_OUT
MODEM_PWR_GATE / MODEM_PWR_OUT
ANNEXE1_OUT / ANNEXE2_OUT
CC1 / CC2 / CC_SENSE
Q6A_3V3_REF / Q6A_PWRKEY / Q6A_SLEEP_REQ / Q6A_HEARTBEAT
SBS_SDA / SBS_SCL
EC25_VIO / TXD / RXD / DTR / RI / PWRKEY
GP6_RESERVE
TP_GP28_3V3 / TP_EC25_RST_OD
```

BQ_INT n'est plus un testpoint obligatoire.

---

# 14. PCB / mécanique

```text
PCB : 4 couches
max strict : 70 x 25 mm
RP2040-Tiny : BOTTOM
```

Stack fonctionnel :

```text
L1 : composants + USB2 + boucles puissance locales
L2 : GND continu
L3 : puissance + signaux lents
L4 : signaux + GND
```

Priorité placement : BQ et boucles de puissance -> USB-C/protection -> MAIN/MODEM -> annexes -> logique/MCU.

---

# 15. BOM principale

## Extended / spécialisés

```text
BQ25628ERYKR       C18221178
XRIM252012S1R0MBCA C22471110
TPS610995DRVR      C2071098
MAKK2016T2R2M      C92923
SN74AVC4T245DR     C22495
JMTQ55P02A         C2890429 x2
TYPE-C-31-M-12     C165948
SRV05-4            C558418
SMF5.0A            C193402
```

## Basic principaux

```text
AO3400A        C20917
AO3401A        C15127
S8050          C2146
B5819W SL      C8598
5.6 kOhm       C23189
5.1 kOhm       C25905
30 kOhm        C22984
4.7 kOhm       C23162
10 kOhm        C25744
100 kOhm       C25741
470 kOhm       C23178
1 MOhm         C22935
100 nF         C1525
1 uF BQ        C52923
1 uF RC hold   C15849
4.7 uF         C1779
10 uF          C15850
47 nF 50 V     C1622
```

Les anciens Micro-Fit sont **supprimés de la BOM**.

Les quantités finales sont générées depuis le schéma V0.6.

---

# 16. Fonctions explicitement supprimées / interdites en V1

```text
USB-C host / DRP / contrôleur CC     supprimé
USB-PD / 9 V                         supprimé
Q6A GPIO59 comme WAKE                remplacé par PWR_ON_KEY ; GPIO59 = HEARTBEAT
CC1/CC2 sur deux ADC                 fusionnés sur GP26
BQ_INT vers RP2040                   supprimé ; polling I2C
Volume +/- vers RP2040               supprimé ; GPIO Q6A directs
EC25 puissance principale par VBUS   supprimée ; BAT depuis MODEM_PWR
USB EC25 à travers PCB Power         supprimé ; faisceau direct
switch USB_VBUS EC25                 non monté
EC25 RESET dédié                     supprimé ; option DNP GP28
DISPLAY_PWR sur carte Power          absent
codec audio Minimal EC25             supprimé
jack utilisateur sur carte Power     absent
mux audio analogique sur carte Power absent
CMC USB2                             non monté
connecteurs Micro-Fit/JST faisceaux  supprimés ; pads THT
```

---

# 17. Gates BLOQUANTES avant fabrication

```text
[ ] Q6A J19 : R7 DNP ; R24 2 mOhm ; R190 100 k ; R191 10 k ; FB4 DNP ; R185..R189 confirmés.
[ ] Q6A J19 : 4.2 / 3.8 / 3.4 / 3.0 V, boot + idle + CPU + vrai deep.
[ ] PWR_ON_KEY : cold boot + wake deep avec J19 final.
[ ] SLEEP_REQ / HEARTBEAT : séquence réelle et aucun faux fault en deep.
[ ] VOL+ GPIO31 / VOL- GPIO30 : événements Linux/Android validés et pas de conflit pinmux.
[ ] GP6 réellement libre dans schéma final et exposé en réserve.
[ ] EC25 VIO/TXD/RI mesurés sur carrier réel.
[ ] EC25 BAT depuis SYS/MODEM_PWR toujours <4.30 V.
[ ] pire cas Q6A charge CPU + EC25 TX LTE + annexes : aucun reset / chute rail.
[ ] USB host EC25 suspend réellement avec VBUS présent.
[ ] MODEM_PWR OFF + VBUS EC25 présent : pas de back-power problématique.
[ ] USB-C branché + MAIN_PWR OFF : pas de back-power Q6A.
[ ] hold MAIN/MODEM : reset MCU réel + Storage OFF + absence de back-power GPIO.
[ ] TPS610995 : shutdown/isolation/courant stockage validés.
[ ] SBS : pull-up Q6A mesurés avant peuplement 4.7 k DNP.
[ ] pack/PCM/fils : courant admissible documenté avec marge.
[ ] ANNEXE1/2 : confirmer que les charges aval acceptent SYS ou disposent de leur régulation.
[ ] footprint RP2040-Tiny 1:1 + FPC accessible.
[ ] layout BQ comparé à la recommandation TI.
[ ] USB-C D+/D- recalculé sur stack-up JLC choisi.
[ ] pads THT puissance dimensionnés et mécaniquement exploitables.
[ ] analyse back-power complète Q6A/EC25/MCU/USB/SBS.
[ ] ERC/DRC propres ; BOM/PnP/polarités/Gerbers revus.
[ ] PCB <=70 x 25 mm sans collision TOP/BOTTOM.
```

Les fonctions radio Android avancées (data complète, SMS, IMS/VoLTE, audio final) ne bloquent pas cette PCB si les interfaces électriques sont correctement prévues.

---

# 18. Tests après fabrication

```text
[ ] USB-C 5 V -> BQ -> SYS sans batterie
[ ] batterie seule -> SYS
[ ] plug/unplug USB sans reboot
[ ] ILIM ~0.45 A au boot
[ ] polling BQ fiable sans INT
[ ] CC_SENSE Default / 1.5 A / 3 A
[ ] IINDPM / EN_EXTILIM corrects
[ ] charge jusqu'à 2 A caractérisée thermiquement
[ ] NTC / télémétrie BQ
[ ] RP stable sur plage SYS
[ ] Storage OFF courant résiduel
[ ] GP6 réserve électriquement libre
[ ] GP28 réserve / RESET OD DNP correct
[ ] MAIN/MODEM OFF par défaut
[ ] reset MCU court : MAIN/MODEM ne chutent pas
[ ] arrêt volontaire : coupure rapide malgré C_HOLD
[ ] Q6A PWRKEY cold boot / wake / long press
[ ] SLEEP_REQ + heartbeat
[ ] VOL+ / VOL- Android natifs
[ ] SBS sans back-power
[ ] USB-C ADB/EDL/MTP
[ ] aucun retour VBUS Q6A vers USB-C
[ ] EC25 BAT dans plage sûre
[ ] EC25 USB direct Q6A stable
[ ] EC25 UART/DTR/RI
[ ] EC25 sleep avec USB suspend
[ ] EC25 PWRKEY / shutdown / recovery
[ ] ANNEXE1 ampli ON/OFF propre sans pop/reboot système excessif
[ ] ANNEXE2 ON/OFF et courant nominal validés
[ ] consommations RUN / deep / storage-off
```

---

# 19. Sources primaires de conception

- Texas Instruments — BQ25628E datasheet / NVDC power-path.
- Texas Instruments — TPS61099x documentation.
- Texas Instruments — SN74AVC4T245 documentation.
- Quectel — EC25 Series Hardware Design V2.4.
- Radxa — Dragon Q6A V1.21 schematic + GPIO/USB documentation.
- Waveshare — RP2040-Tiny schematic/mécanique.
- JLCPCB/LCSC — bibliothèque et règles de fabrication.

---

# 20. Gel proposé

Après validation utilisateur de ce document, les éléments suivants sont considérés **gelés pour le schéma/PCB V0.6** :

```text
batterie 1S / BQ25628E / SYS
USB-C 5 V device-only
MAIN_PWR et MODEM_PWR séparés avec hold RC 1 MOhm + 1 uF
PWR_ON_KEY réel Q6A
GPIO58 SLEEP_REQ
GPIO59 HEARTBEAT
SBS GP14/GP15
BQ sans INT MCU ; polling I2C ; GP6 réserve
GP28 réserve + RESET_N OD DNP
VOL+ = Q6A GPIO31 / J20 pin 29
VOL- = Q6A GPIO30 / J20 pin 32
ANNEXE1 = ampli haut-parleur
ANNEXE2 = SYS commuté générique annexes
aucun connecteur faisceau : pastilles THT pour fils soudés
USB Q6A <-> EC25 hors PCB Power
```

Toute modification ultérieure d'un de ces choix doit être traitée comme une révision d'architecture ou une ECO explicite, pas comme une correction silencieuse du schéma.