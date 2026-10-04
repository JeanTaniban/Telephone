# CDC — Carte Power / MCU V1 du Maker Phone Q6A — CURRENT

**Date :** 2026-10-04  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **VALIDÉ / GELÉ — schéma/PCB V0.7**  
**Fabrication cible :** JLCPCB Economic PCBA  
**Priorité composants :** Basic > Promotional > Extended ; stock et classification à revalider au moment de la commande.

Ce document est la **référence électrique normative courante** de la carte Power/MCU. Il supersède les anciens CDC Power/MCU dès qu'ils sont contradictoires.

Le gel V0.7 intègre toutes les décisions prises pendant la revue d'architecture, notamment :

```text
- batterie cible fixée : Motorola JK50 1S, Genuine Service Pack ;
- connecteur batterie PCB fixé : Molex 5050060812 / LCSC C779875 ;
- pinout JK50 documenté : 1/8 GND, 4/5 VBAT, 2 ID2, 3 ID1, 6 NTC1, 7 NTC2 ;
- BAT_ARM mécanique ajouté entre batterie et BQ afin de brancher la batterie carte désarmée ;
- pas de deuxième BMS/PCM en série sur le PCB ;
- pas de fusible batterie obligatoire en baseline V1 ;
- pas d'anti-inversion série en baseline ; le connecteur batterie est détrompé ;
- JK50 HV ~4.40 V volontairement sous-chargée : BQ VREG initial ~4.15 V ;
- BQ25628E INT non relié au MCU ; surveillance par polling I2C ;
- GP6 réserve ; GP28 réserve + option EC25 RESET_N open-drain DNP ;
- Volume+ / Volume- directement sur deux GPIO du Q6A, pas sur le MCU ;
- VOL+ = Q6A J20 pin 29 / GPIO31 ;
- VOL- = Q6A J20 pin 32 / GPIO30 ;
- ANNEXE1 = alimentation ampli haut-parleur ;
- ANNEXE2 = rail SYS commuté générique pour caméras / flash / outils / annexes ;
- hors batterie, aucun connecteur de faisceau sur la carte Power : pastilles THT pour fils soudés ;
- MAIN_PWR, MODEM_PWR, PWR_ON_KEY, SLEEP_REQ, HEARTBEAT, SBS et supervision EC25 conservés.
```

---

# 0. Architecture finale retenue

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
        +--> VBUS --> TVS --> BQ25628E --> SYS -----------------------------+
        |                         |                                          |
        |                         +<------> BAT                               |
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

# 1. Batterie JK50, connecteur, protection et BAT_ARM

## 1.1 Modèle de batterie figé

La batterie V1 est une **Motorola JK50** de préférence **Genuine Service Pack** (famille utilisée notamment sur plusieurs Moto G/E).

Caractéristiques de référence retenues pour le CDC :

```text
Référence famille      : Motorola JK50
Chimie / architecture  : Li-ion/LiPo pouch 1S smartphone
Tension nominale       : ~3.80 V
Capacité rated         : ~4850 mAh
Capacité typical       : ~5000 mAh
Énergie                : ~18.5 à 19 Wh selon révision
Tension charge pack    : jusqu'à ~4.40 V pour le pack d'origine
Charge max documentée  : ~3.0 A
Décharge max documentée: ~4.85 A
```

La JK50 est donc une **cellule HV**. C'est volontaire et accepté : la carte V1 **ne la charge pas jusqu'à 4.40 V**.

Le BQ25628E est configuré initialement autour de :

```text
VREG = 4.15 V
ICHG <= 2 A
```

Conséquences :

```text
- la cellule est volontairement sous-chargée ;
- marge accrue vis-à-vis du domaine EC25 alimenté depuis SYS ;
- capacité utile réelle inférieure aux ~5000 mAh annoncés à pleine charge 4.40 V ;
- le SOC Android doit représenter la fenêtre réellement utilisée par notre téléphone,
  pas la capacité électrochimique complète de la JK50.
```

Ne jamais augmenter VREG vers 4.40 V sans réétudier complètement MODEM_PWR / EC25 et les transitoires SYS.

## 1.2 Dimensions batterie

Dimensions de référence d'intégration pour la famille JK50 :

```text
~86.1 x 65.0 x 4.8 mm
```

Réservation mécanique recommandée au stade prototype :

```text
>= 87 x 66 x 5.5 mm
```

Cette réservation laisse une petite marge de tolérance mais **ne remplace pas la mesure de la vraie Genuine Service Pack reçue**, notamment autour du flex, de la protection du pack et de la sortie connecteur.

Avant gel mécanique du châssis : mesurer largeur, longueur, épaisseur, position du flex, rayon de courbure et hauteur du connecteur sur l'exemplaire réel.

## 1.3 Connecteur batterie figé

Connecteur côté **notre PCB** :

```text
J_BAT        : Molex 5050060812
LCSC/JLC ref : C779875
Nombre pins  : 8
Pas          : 0.4 mm
Hauteur mate : ~0.75 mm
Famille      : Molex Battery / board-to-board
```

Connecteur complémentaire côté flex batterie :

```text
Molex 5050040812
```

La série est conçue avec **4 contacts puissance + 4 contacts signal** et est adaptée à plusieurs ampères ; le couple est cohérent avec la JK50 et ses ~4.85 A max documentés.

Le `C779875` est un composant spécialisé/Extended à accepter en V1. Le stock JLC/LCSC doit être revalidé au moment de la commande. Si JLC ne peut pas l'assembler, le plan de secours est assemblage manuel/rework spécialisé ; le footprint reste celui du Molex officiel.

**Le footprint et l'orientation doivent provenir exclusivement du drawing Molex officiel.** Ne jamais déduire le pin 1 à partir d'une photo de batterie.

## 1.4 Pinout JK50 / Molex 5050060812

Brochage retenu, recoupé sur les schémas de réparation Motorola utilisant cette famille :

```text
Pin 1 -> BATT- / GND
Pin 2 -> ID2
Pin 3 -> ID1
Pin 4 -> VBAT+
Pin 5 -> VBAT+
Pin 6 -> NTC1
Pin 7 -> NTC2
Pin 8 -> BATT- / GND
```

Câblage PCB :

```text
pins 4 + 5 -> BAT_RAW+
pins 1 + 8 -> GND / BAT-
pin 2      -> TP_BAT_ID2
pin 3      -> TP_BAT_ID1
pin 6      -> TP_BAT_NTC1 + sélection vers BQ_TS
pin 7      -> TP_BAT_NTC2 + sélection alternative vers BQ_TS
```

Les lignes ID1/ID2 ne sont pas nécessaires au fonctionnement V1 du téléphone et restent en haute impédance/testpoints tant qu'une fonction n'est pas justifiée.

## 1.5 NTC1 / NTC2

La JK50 expose deux lignes thermiques. Les documents Motorola permettent d'identifier `NTC1` et `NTC2`, mais **la courbe R/T exacte du thermistor du pack doit être caractérisée sur la vraie batterie reçue** avant de figer la population TS du BQ.

Le PCB doit donc prévoir :

```text
NTC1 -> TP -> strap 0 Ohm / DNP -> réseau BQ_TS
NTC2 -> TP -> strap 0 Ohm / DNP -> réseau BQ_TS
```

Une seule source thermique est sélectionnée vers BQ_TS à la fois.

Conserver si la place est gratuite deux pads `NTC_EXT / GND` permettant de coller un NTC externe 10 kOhm sur la batterie en solution de secours.

L'ancien réseau calculé pour un **Semitec 103AT-2 10 kOhm** reste une référence/fallback mais **n'est plus supposé compatible avec la JK50 sans mesure**.

Les résistances `RT1/RT2` du réseau TS doivent donc être considérées comme **population à valider après mesure JK50**, pas comme une hypothèse irrévocable.

## 1.6 BAT_ARM — armement batterie

Objectif : pouvoir brancher/remplacer la batterie sans mettre immédiatement toute la carte sous tension.

Topology :

```text
J_BAT pins 4/5 -> BAT_RAW+
                   |
             PAD_BAT_ARM_A
                   |
             interrupteur SPST externe
                   |
             PAD_BAT_ARM_B
                   |
                  BAT
                   |
             BQ25628E BAT
```

Exigences :

```text
- 2 grosses pastilles THT clairement marquées ;
- interrupteur externe DC basse tension, >=5 A avec faible résistance de contact ;
- ouvert pendant branchement/débranchement batterie ;
- fermé seulement après inspection / mesure ;
- chemin cuivre BAT_RAW/BAT dimensionné pour le courant pack.
```

`BAT_ARM` coupe **la batterie positive**, pas le port USB-C.

Important :

```text
BAT_ARM ouvert + USB-C absent  -> carte non alimentée par la batterie
BAT_ARM ouvert + USB-C présent -> le BQ peut alimenter SYS depuis VBUS
```

`BAT_ARM` n'est donc pas un interrupteur général absolu si l'USB-C est branché.

## 1.7 BAT_ARM versus STORAGE_SW

Les deux interrupteurs ont des rôles distincts :

```text
BAT_ARM
  -> isole physiquement la batterie du BQ / de la carte
  -> utile assemblage, maintenance, stockage profond et sécurité de manipulation

STORAGE_SW
  -> coupe seulement l'alimentation du RP2040 via EN du TPS610995
  -> n'isole pas la batterie du BQ/SYS
  -> utilisé après shutdown normal lorsque MAIN_PWR et MODEM_PWR sont déjà OFF
```

Il ne faut pas remplacer l'un par l'autre.

## 1.8 PCM/BMS, fusible et anti-inversion

La JK50 est utilisée comme **pack smartphone protégé**. Pour une batterie 1S il n'y a pas d'équilibrage multi-cellules ; la protection interne est assimilée ici au **PCM/protection pack**.

Architecture V1 :

```text
JK50 avec protection pack
        |
     BAT_ARM
        |
   BQ25628E
        |
       SYS
```

Décisions :

```text
- aucun second BMS/PCM complet en série sur notre PCB ;
- BQ25628E = chargeur/power-path, pas remplacement du PCM batterie ;
- aucun fusible batterie obligatoire en baseline V1 ;
- aucun PTC obligatoire ;
- aucune diode Schottky série sur BAT ;
- aucun P-MOS anti-inversion série en baseline.
```

Raisons :

```text
- un second BMS ajoute résistance, seuils de coupure concurrents et recovery ambigu ;
- une Schottky est trop dissipative sur un système 1S à plusieurs ampères ;
- un MOS supplémentaire ajoute perte et complexité dans le chemin de tous les pics ;
- le connecteur Molex batterie est détrompé et interne au téléphone ;
- le pack possède sa propre protection.
```

Un footprint de fusible DNP n'est **pas requis**. Il peut être ajouté uniquement si son coût mécanique/routage est nul ; il ne doit pas être une dépendance de la V1.

Le seuil OCP exact de la protection JK50 n'est pas considéré comme publiquement garanti dans le CDC. Il doit être caractérisé indirectement au banc ; ne jamais supposer que `4.85 A` est le seuil de déclenchement du PCM. `4.85 A` est la décharge maximale documentée du pack.

## 1.9 Budget courant batterie

La JK50 est acceptée malgré une décharge max documentée proche de **4.85 A**, donc la marge sur le pire cas système n'est pas énorme.

Gate obligatoire :

```text
Q6A charge CPU/GPU + EC25 TX LTE + ANNEXE1/ANNEXE2 selon scénario
-> aucune coupure pack
-> aucune chute VBAT dangereuse
-> aucun reset Q6A/EC25
-> température batterie acceptable
```

Ne pas supposer que ANNEXE1 et ANNEXE2 peuvent fournir chacun 1 A simultanément avec le pire cas Q6A + LTE. Si nécessaire, le firmware imposera une politique de puissance : coupe ANNEXE2, réduit la charge, ou évite certaines combinaisons lors des pics radio.

---

# 2. BQ25628E — chargeur / power-path

## 2.1 Référence

```text
U_CHG       : BQ25628ERYKR
JLC/LCSC    : C18221178
Boîtier     : WQFN-18 2.5 x 3 mm
Batterie    : 1S
Charge max  : 2 A dans notre politique V1
Power-path  : NVDC
ADC/I2C     : oui
```

Le BQ fournit VBUS/VBAT/VSYS/IBUS/IBAT/TS et les états charge/fault. Il n'est pas considéré comme fuel-gauge de précision.

Il doit fonctionner de manière sûre même si le RP2040 est bloqué ou éteint.

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

**GP6 n'est pas consommé par BQ_INT.**

Le MCU surveille le BQ par polling I2C. Les protections du BQ ne doivent pas dépendre d'une interruption MCU.

Politique firmware initiale :

```text
RUN / charge active : polling typique ~1 s
veille MCU          : cadence ralentie selon besoin
réveil / événement  : lecture immédiate de tous les états utiles
```

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

`VREG=4.15 V` est une politique volontairement conservatrice avec la JK50 HV ; elle n'essaie pas de récupérer les 5000 mAh complets annoncés à la tension haute du pack.

## 2.4 MODEM_PWR et plage SYS

Quectel EC25 : domaine d'alimentation à maintenir dans la plage sûre du module, avec absolu à respecter.

Politique de départ :

```text
MODEM_PWR autorisé si environ 3.35 V <= VSYS <= 4.25 V
```

Gate absolue :

```text
EC25 carrier BAT doit rester < 4.30 V dans tous les états,
y compris batterie chargée, USB branché et transitoires.
```

En dessous d'environ 3.35 V, tenter un arrêt propre du modem avant `MODEM_PWR OFF`.

## 2.5 TS / thermique

La topologie BQ_TS est conservée mais devient **configurable pour la JK50**.

Anciennes valeurs de référence pour un NTC externe 10 kOhm 103AT-2 :

```text
RT1 = 5.1 kOhm / C25905 Basic
RT2 = 30 kOhm  / C22984 Basic
```

Ces valeurs ne sont à peupler pour la JK50 que si la caractérisation NTC confirme la compatibilité. Sinon recalculer la population sans modifier le PCB.

Validation thermique obligatoire avant d'autoriser 2 A de charge.

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
- vias multiples sur changements de couche puissance ;
- J_BAT et BAT_ARM placés de manière à garder BAT_RAW/BAT court et robuste.
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

Le host utilisateur futur passe par un port USB2 host Q6A dédié, pas par l'USB-C extérieur.

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

La Schottky empêche un retour simple du 5 V Q6A vers l'entrée USB-C/BQ.

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

Une seule CC est active selon orientation. Le point ADC voit environ la moitié de la tension CC active.

Firmware : attendre stabilisation, moyenner, et traiter tout niveau ambigu comme source Default/conservatrice.

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

Le FPC du module reste le chemin USB/BOOTSEL/RUN de récupération. Pas de SWD supplémentaire obligatoire en V1.

## 4.2 Alimentation

```text
SYS -> TPS610995DRVR / C2071098 -> MCU_VSYS -> VSYS RP2040-Tiny
L_MCU : MAKK2016T2R2M / C92923 / 2.2 uH
CIN   : 10 uF
COUT  : 2 x 10 uF
```

Le rail s'appelle `MCU_VSYS` car le TPS610995 peut fonctionner en pass-through selon VIN.

## 4.3 Storage switch

```text
SYS -> STORAGE_SW -> EN TPS610995
EN -> 100 kOhm -> GND
```

```text
fermé  : MCU alimenté
ouvert : hard-off MCU / stockage
```

Storage OFF ne remplace jamais le shutdown normal et ne remplace pas BAT_ARM.

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

**Deux réserves pratiques sont conservées : GP6 et GP28.**

GP6 doit être exposé simplement sans fonction imposée. GP28 conserve en plus l'étage RESET_N optionnel DNP décrit plus bas.

---

# 5. MAIN_PWR et MODEM_PWR — OFF par défaut + maintien reset MCU

## 5.1 Puissance

```text
Q_MAIN_P  : JMTQ55P02A / C2890429
Q_MODEM_P : JMTQ55P02A / C2890429
P-MOS 20 V, faible RDS(on) à faible VGS
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

Constante RC nominale ~1 s ; le délai réel dépend du seuil AO3400A.

Cible mesurée :

```text
~0.7 à 1.5 s
```

```text
GPIO HIGH -> ON
GPIO LOW  -> OFF rapide
GPIO Hi-Z -> maintien temporaire par RC
```

Le condensateur est sur la grille du petit NMOS, pas sur le P-MOS puissance.

Au boot MCU, GP10 et GP29 sont traités en priorité absolue pour réaffirmer HIGH avant expiration du hold lorsqu'un rail était déjà actif.

---

# 6. ANNEXE1 / ANNEXE2

Branches simples, **OFF par défaut**, sans RC de maintien.

```text
P-MOS : AO3401A / C15127 Basic
cible électrique locale : ~1 A continu max par annexe,
sous réserve thermique/layout et budget courant batterie global
```

## 6.1 ANNEXE1

Fonction figée : **alimentation commutée de l'ampli haut-parleur**.

```text
GP11 -> ANNEXE1_EN
SYS -> ANNEXE1_OUT -> ampli
```

Le signal audio lui-même reste géré par le Q6A / chaîne audio dédiée, hors fonction Power de cette carte.

Le firmware peut couper l'ampli pendant certaines phases radio/appel si cela réduit bruit ou consommation.

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

ANNEXE2 est du **SYS commuté non régulé**. Toute annexe nécessitant 5 V, 3.3 V fixe ou une autre tension doit posséder son propre convertisseur/régulateur aval.

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

J19 devient l'entrée 1S principale du Q6A depuis MAIN_PWR.

## 7.3 PWR_ON_KEY

```text
GP8 -> 10 kOhm -> base S8050
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Fonctions :

```text
- cold boot après MAIN_PWR ON ;
- wake depuis deep ;
- événement Power Android ;
- appui long de récupération si nécessaire.
```

Pulse initial à caractériser autour de 100–300 ms.

## 7.4 SLEEP_REQ

```text
Q6A_3V3 -> 100 kOhm -> SLEEP_REQ
SLEEP_REQ -> 10 kOhm série -> GP9
```

MCU : actif = sortie LOW ; repos = input/Hi-Z.

SLEEP_REQ reste distinct de PWR_ON_KEY car le suspend normal doit exécuter une procédure propre : EC25, USB host, OTG externe, Wi-Fi/wake sources, puis `mem_sleep=deep`.

## 7.5 HEARTBEAT

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Le heartbeat est un **statut logiciel**, pas une sécurité combinatoire.

Il peut coder RUN / transition / shutdown et peut s'arrêter volontairement en deep sleep.

Sa perte seule ne coupe jamais MAIN_PWR immédiatement. Le MCU tente d'abord récupération/wake, puis seulement un power-cycle après timeout explicite.

## 7.6 SBS / batterie virtuelle Android

```text
GP14 <-> Q6A I2C6_SDA
GP15 <-> Q6A I2C6_SCL
```

Prévoir :

```text
2 x 4.7 kOhm vers Q6A_3V3, DNP par défaut
```

Ne jamais tirer SBS vers 3V3_MCU afin d'éviter le back-power du Q6A éteint.

Mesurer d'abord les pull-up déjà présents sur le Q6A.

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

Cette décision conserve GP6 et GP28 disponibles et évite toute dépendance MCU pour le volume Android.

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

Gates :

```text
- le host Q6A doit suspendre réellement l'USB en deep ;
- VBUS présent avec MODEM_PWR OFF ne doit pas back-powerer significativement l'EC25 ;
- si nécessaire, le faisceau direct pourra être modifié plus tard pour interrompre VBUS,
  sans refaire la carte Power.
```

## 9.3 UART / DTR / RI

```text
SN74AVC4T245DR / C22495
VCCA = EC25 VIO (~1.8 V)
VCCB = 3V3_MCU

EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

Le composant est intégralement occupé par ces quatre signaux.

Prévoir :

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

Recovery :

```text
AT propre -> PWRKEY -> MODEM_PWR power-cycle
```

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

## 10.1 États logiques

```text
BAT_DISARMED : BAT_ARM ouvert ; batterie isolée ; USB-C peut néanmoins alimenter SYS
STORAGE      : après shutdown, MCU hard-off par STORAGE_SW ; BAT_ARM peut rester fermé
OFF          : MCU ON ; MAIN/MODEM OFF
BOOT         : MAIN ON -> délai PMIC -> PWR_ON_KEY -> heartbeat -> MODEM si SYS sûr
RUN          : MAIN ON ; fonctionnement normal
DEEP         : MAIN ON ; MODEM éventuellement ON ; heartbeat peut s'arrêter volontairement
SHUTDOWN     : Android + modem arrêtés proprement -> MODEM OFF -> MAIN OFF
FAULT        : récupération graduelle ; hard cut seulement en dernier recours
```

## 10.2 Procédure de première mise sous tension / maintenance

```text
1. USB-C débranché.
2. BAT_ARM ouvert.
3. brancher JK50 sur J_BAT.
4. vérifier polarité mécanique, absence de court BAT_RAW-GND et tension BAT_RAW.
5. fermer BAT_ARM.
6. MCU démarre si STORAGE_SW fermé.
7. système reste en état OFF jusqu'à demande de boot Q6A.
```

## 10.3 Cold boot

```text
BAT_ARM fermé
STORAGE_SW fermé
-> MCU boot
-> lecture/config BQ
-> MAIN_PWR_EN HIGH
-> attendre stabilisation J19 / PMIC
-> pulse Q6A PWR_ON_KEY
-> attendre HEARTBEAT
-> si VSYS sûr : MODEM_PWR_EN HIGH
-> pulse EC25 PWRKEY
```

## 10.4 Reset MCU pendant RUN

```text
RC MAIN/MODEM maintient ~1 s
-> RP2040 reboot
-> GP10/GP29 réaffirmés immédiatement
-> pas de reset Q6A/modem attendu
-> resynchronisation HEARTBEAT/BQ/UART
```

## 10.5 Suspend normal

```text
Power utilisateur -> MCU
-> SLEEP_REQ
-> Q6A prépare :
   EC25 QSCLK/DTR + USB host suspend
   USB OTG externe -> none si requis
   Wi-Fi/autres wake sources traités
   heartbeat adapté
-> mem_sleep=deep
```

Réveil : pulse `Q6A PWR_ON_KEY`.

MAIN_PWR reste ON. MODEM_PWR reste ON si le modem doit demeurer joignable.

## 10.6 Shutdown normal

```text
1. shutdown Android propre
2. arrêt EC25 propre
3. timeout / confirmation
4. MODEM_PWR OFF
5. MAIN_PWR OFF
6. MCU reste ON tant que STORAGE_SW est fermé
```

Après shutdown, on peut :

```text
- ouvrir STORAGE_SW pour hard-off MCU ;
- ou ouvrir BAT_ARM pour isoler physiquement la batterie.
```

---

# 11. SOC / batterie Android

Le MCU utilise :

```text
VBAT / VSYS / IBAT / IBUS / TS
état charge/faults
courbe OCV/SOC adaptée à la JK50
intégration logicielle lorsque pertinente
```

Exposition via SBS possible.

Le SOC V1 représente la **fenêtre d'énergie réellement utilisée avec VREG ~4.15 V**, pas nécessairement les ~5000 mAh accessibles lorsque la JK50 est chargée jusqu'à sa tension HV d'origine.

Ce n'est pas un fuel-gauge de précision.

`MAX17048` n'est pas requis. Un footprint DNP éventuel ne doit jamais être une dépendance V1.

---

# 12. Connectique, faisceaux et pastilles THT

## 12.1 Exception batterie

La batterie est la **seule liaison interne disposant d'un connecteur PCB dédié** :

```text
J_BAT = Molex 5050060812 / C779875
mate pack = Molex 5050040812
```

Cela évite de souder directement une batterie active sur la carte.

## 12.2 Règle générale hors batterie

**Aucun connecteur de faisceau n'est retenu pour Q6A, EC25, annexes, boutons ou interrupteurs.**

Ces liaisons utilisent des pastilles traversantes percées, dimensionnées pour soudure manuelle et clairement sérigraphiées.

Groupes minimum :

```text
BAT_ARM  : BAT_RAW+ / BAT (2 grosses pastilles vers interrupteur externe)
BAT AUX  : ID1 / ID2 / NTC1 / NTC2 en TP/straps ; BAT_RAW+/GND service si utile
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

Les sorties de puissance et BAT_ARM utilisent des trous/pads plus gros que les signaux logiques.

Prévoir soulagement mécanique / fixation de faisceaux hors pads quand la mécanique finale le permet ; les pads ne doivent pas reprendre seuls les efforts d'arrachement.

---

# 13. Testpoints obligatoires

```text
VBUS_USB_C / BQ_VBUS
BAT_RAW / BAT / SYS / GND
BAT_ID1 / BAT_ID2 / BAT_NTC1 / BAT_NTC2 / BQ_TS
MCU_VSYS / 3V3_MCU / TPS610995_EN
BQ_SDA / BQ_SCL / BQ_ILIM / BQ_PG
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

Priorité placement :

```text
1. BQ + ses boucles puissance
2. J_BAT + BAT_ARM + chemin BAT
3. USB-C + protections
4. MAIN_PWR / MODEM_PWR
5. annexes
6. logique / RP2040
```

Contraintes batterie/connecteur :

```text
- accès mécanique au connecteur JK50 ;
- aucune collision avec flex batterie ;
- drawing Molex respecté ;
- sérigraphie pin 1 et BAT+ clairement visible ;
- BAT_RAW et GND espacés et protégés des outils lors du montage ;
- réservation châssis batterie >=87 x 66 x 5.5 mm provisoire,
  à confirmer sur pack réel.
```

---

# 15. BOM principale

## 15.1 Extended / spécialisés

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
Molex 5050060812   C779875     J_BAT batterie JK50
```

Le stock et le statut JLC du Molex doivent être revalidés juste avant commande.

## 15.2 Basic principaux

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

Les valeurs 5.1 kOhm / 30 kOhm liées à TS sont à confirmer pour la JK50 ; conserver les footprints même si la valeur finale change.

Les anciens Micro-Fit/JST de faisceau sont supprimés de la BOM.

Les quantités finales sont générées depuis le schéma V0.7.

---

# 16. Fonctions explicitement supprimées / interdites en V1

```text
USB-C host / DRP / contrôleur CC        supprimé
USB-PD / 9 V                            supprimé
Q6A GPIO59 comme WAKE                   remplacé par PWR_ON_KEY ; GPIO59 = HEARTBEAT
CC1/CC2 sur deux ADC                    fusionnés sur GP26
BQ_INT vers RP2040                      supprimé ; polling I2C
Volume +/- vers RP2040                  supprimé ; GPIO Q6A directs
EC25 puissance principale par VBUS      supprimée ; BAT depuis MODEM_PWR
USB EC25 à travers PCB Power            supprimé ; faisceau direct
switch USB_VBUS EC25                    non monté
EC25 RESET dédié                        supprimé ; option DNP GP28
DISPLAY_PWR sur carte Power             absent
codec audio Minimal EC25                supprimé
jack utilisateur sur carte Power        absent
mux audio analogique sur carte Power    absent
CMC USB2                                non monté
connecteurs Micro-Fit/JST faisceaux     supprimés ; pads THT
2e BMS/PCM batterie sur PCB             supprimé
fusible batterie obligatoire            supprimé de la baseline
Schottky série anti-inversion BAT        supprimée
P-MOS anti-inversion BAT                non monté en baseline
charge JK50 à 4.40 V                    interdite en V1 ; VREG initial ~4.15 V
```

Le connecteur Molex `J_BAT` est l'exception volontaire à la règle « pas de connecteurs de faisceau ».

---

# 17. Gates BLOQUANTES avant fabrication

## 17.1 Batterie / J_BAT

```text
[ ] Acheter une vraie Motorola JK50 Genuine Service Pack.
[ ] Mesurer dimensions réelles + position/longueur flex avant gel mécanique châssis.
[ ] Vérifier physiquement le mating JK50 <-> Molex 5050060812.
[ ] Vérifier au multimètre sur le pack réel : pins 1/8 GND ; 4/5 VBAT ; 2 ID2 ; 3 ID1 ; 6 NTC1 ; 7 NTC2.
[ ] Caractériser NTC1 et NTC2 à plusieurs températures raisonnables.
[ ] Choisir NTC1 ou NTC2 vers BQ_TS et figer RT1/RT2 après caractérisation.
[ ] Vérifier absence de consommation/fonction indésirable sur ID1/ID2 laissés en haute impédance.
[ ] Vérifier footprint Molex avec drawing officiel + impression 1:1.
[ ] Revalider stock JLC/LCSC C779875 avant commande.
[ ] BAT_ARM : switch externe >=5 A, faible R de contact, pads et cuivre adaptés.
[ ] JK50 pire cas système : pas de coupure pack / reset jusqu'au scénario de charge maximal réaliste.
```

## 17.2 Q6A / modem / système

```text
[ ] Q6A J19 : R7 DNP ; R24 2 mOhm ; R190 100 k ; R191 10 k ; FB4 DNP ; R185..R189 confirmés.
[ ] Q6A J19 : 4.2 / 3.8 / 3.4 / 3.0 V, boot + idle + CPU + vrai deep.
[ ] PWR_ON_KEY : cold boot + wake deep avec J19 final.
[ ] SLEEP_REQ / HEARTBEAT : séquence réelle et aucun faux fault en deep.
[ ] VOL+ GPIO31 / VOL- GPIO30 : événements Linux/Android validés et pas de conflit pinmux.
[ ] GP6 réellement libre dans schéma final et exposé en réserve.
[ ] EC25 VIO/TXD/RI mesurés sur carrier réel.
[ ] EC25 BAT depuis SYS/MODEM_PWR toujours <4.30 V.
[ ] pire cas Q6A charge CPU + EC25 TX LTE + annexes : aucun reset / chute rail / déclenchement protection JK50.
[ ] USB host EC25 suspend réellement avec VBUS présent.
[ ] MODEM_PWR OFF + VBUS EC25 présent : pas de back-power problématique.
[ ] USB-C branché + MAIN_PWR OFF : pas de back-power Q6A.
[ ] hold MAIN/MODEM : reset MCU réel + Storage OFF + absence de back-power GPIO.
[ ] TPS610995 : shutdown/isolation/courant stockage validés.
[ ] SBS : pull-up Q6A mesurés avant peuplement 4.7 k DNP.
[ ] ANNEXE1/2 : charges aval acceptent SYS ou disposent de leur régulation.
[ ] footprint RP2040-Tiny 1:1 + FPC accessible.
[ ] layout BQ comparé à la recommandation TI.
[ ] USB-C D+/D- recalculé sur stack-up JLC choisi.
[ ] pads THT puissance dimensionnés et mécaniquement exploitables.
[ ] analyse back-power complète Q6A/EC25/MCU/USB/SBS/batterie.
[ ] ERC/DRC propres ; BOM/PnP/polarités/Gerbers revus.
[ ] PCB <=70 x 25 mm sans collision TOP/BOTTOM.
```

Les fonctions radio Android avancées (data complète, SMS, IMS/VoLTE, audio final) ne bloquent pas cette PCB si les interfaces électriques sont correctement prévues.

---

# 18. Tests après fabrication

## 18.1 Batterie / armement

```text
[ ] BAT_ARM ouvert + JK50 branchée + USB absent : aucun rail alimenté depuis BAT.
[ ] BAT_RAW correspond à la tension pack.
[ ] BAT_ARM fermeture : BAT rejoint BAT_RAW sans chute anormale.
[ ] BAT_ARM ouvert + USB-C présent : comportement attendu, SYS peut être alimenté depuis VBUS.
[ ] J_BAT : aucun échauffement / faux contact aux pics réalistes.
[ ] NTC1/NTC2 cohérents avec température pack.
[ ] charge JK50 limitée à VREG programmé ~4.15 V.
[ ] ICHG <=2 A et thermique pack/BQ acceptable.
[ ] décharge forte réaliste : aucune coupure de protection pack.
```

## 18.2 Power / USB / MCU / modem

```text
[ ] USB-C 5 V -> BQ -> SYS sans batterie
[ ] batterie seule -> SYS
[ ] plug/unplug USB sans reboot
[ ] ILIM ~0.45 A au boot
[ ] polling BQ fiable sans INT
[ ] CC_SENSE Default / 1.5 A / 3 A
[ ] IINDPM / EN_EXTILIM corrects
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
[ ] consommations RUN / deep / storage-off / BAT_ARM ouvert
```

---

# 19. Sources primaires / documents de référence

Conception électronique :

- Texas Instruments — BQ25628E datasheet / NVDC power-path.
- Texas Instruments — TPS61099x documentation.
- Texas Instruments — SN74AVC4T245 documentation.
- Quectel — EC25 Series Hardware Design V2.4.
- Radxa — Dragon Q6A V1.21 schematic + GPIO/USB documentation.
- Waveshare — RP2040-Tiny schematic/mécanique.
- JLCPCB/LCSC — bibliothèque et règles de fabrication.

Batterie JK50 / connecteur :

- Motorola Level-3 / repair schematics des modèles utilisant JK50, pour J_BAT et pinout ID/NTC/VBAT/GND.
- Molex — série 505006 / 505004, drawing officiel et mating.
- Documentation transport/UN38.3 de la famille JK50 pour tension/capacité/courants de référence.
- Mesures sur la **vraie JK50 Genuine Service Pack** obligatoires pour dimensions finales et caractérisation NTC.

---

# 20. Gel V0.7

Les choix suivants sont **gelés pour le schéma/PCB V0.7** :

```text
BATTERIE
- Motorola JK50 1S, cible Genuine Service Pack
- ~4850 mAh rated / ~5000 mAh typical
- ~3.80 V nominal ; pack HV capable ~4.40 V mais sous-chargé en V1
- dimensions de travail ~86.1 x 65.0 x 4.8 mm ; pack réel à mesurer
- connecteur PCB Molex 5050060812 / C779875
- mate batterie Molex 5050040812
- pinout : 1/8 GND ; 4/5 VBAT ; 2 ID2 ; 3 ID1 ; 6 NTC1 ; 7 NTC2
- sélection NTC1/NTC2 configurable vers BQ_TS
- BAT_ARM mécanique sur BAT+ avant BQ
- pas de second BMS/PCM PCB
- pas de fusible obligatoire baseline
- pas d'anti-inversion série baseline

CHARGE / SYS
- BQ25628E / SYS NVDC
- VREG initial ~4.15 V
- ICHG <=2 A
- USB-C 5 V device-only
- CC1/CC2 fusionnés vers GP26 ADC

Q6A / EC25
- MAIN_PWR et MODEM_PWR séparés avec hold RC 1 MOhm + 1 uF
- PWR_ON_KEY réel Q6A
- GPIO58 SLEEP_REQ
- GPIO59 HEARTBEAT
- SBS GP14/GP15
- USB Q6A <-> EC25 hors PCB Power
- EC25 UART/DTR/RI/PWRKEY conservés

MCU / GPIO
- BQ sans INT MCU ; polling I2C
- GP6 réserve
- GP28 réserve + RESET_N EC25 open-drain DNP
- VOL+ = Q6A GPIO31 / J20 pin 29
- VOL- = Q6A GPIO30 / J20 pin 32

ANNEXES / MÉCANIQUE
- ANNEXE1 = ampli haut-parleur
- ANNEXE2 = SYS commuté générique annexes
- J_BAT est le seul connecteur interne dédié sur la carte
- toutes les autres liaisons internes = pastilles THT pour fils soudés
- PCB 4 couches <=70 x 25 mm
- RP2040-Tiny BOTTOM
```

Les éléments encore à **caractériser** (NTC JK50, tenue pire cas 4.85 A, dimensions réelles du pack, back-power, hold RC réel, etc.) sont des **gates de validation physique**, pas des choix d'architecture ouverts.

Toute modification ultérieure d'un élément gelé doit être traitée comme une révision d'architecture ou une ECO explicite, jamais comme une correction silencieuse du schéma.