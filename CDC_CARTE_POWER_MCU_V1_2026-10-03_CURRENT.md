# CDC — Carte Power / MCU V1 du Maker Phone Q6A — CURRENT

**Date :** 2026-10-03  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **CDC électrique courant pour schéma/PCB V0.5**  
**Fabrication cible :** JLCPCB Economic PCBA  
**Priorité composants :** Basic > Promotional > Extended ; revalider stock/classification au moment de la commande.

Ce document **supersède pour les choix Power/MCU** :

```text
CDC_CARTE_POWER_MCU_V1_2026-09-30.md
CDC_CARTE_POWER_MCU_V1_2026-10-02_REVIEW_FINAL.md
```

Le document du 02/10 reste l'audit de travail. La présente version intègre les décisions utilisateur finales prises ensuite, notamment le vrai `PWR_ON_KEY`, le heartbeat, le maintien RC validé à `1 MOhm + 1 uF`, la fusion CC, la réserve GP28 et l'architecture MODEM_PWR.

---

# 0. Architecture finale retenue

```text
USB-C extérieur 5 V / USB2 DEVICE uniquement
        |
        +--> VBUS --> TVS --> BQ25628E --> SYS --------------------------+
        |                         |                                       |
        |                         +--> BAT --> batterie 1S protégée + NTC |
        |                                                                 |
        +--> D+/D- -----------------------------> Q6A USB OTG DEVICE      |
        +--> VBUS -- Schottky ------------------> Q6A OTG VBUS            |
                                                                          |
SYS ----------------------------------------------------------------------+----+
 |                                                                             |
 +--> TPS610995 --> MCU_VSYS --> VSYS RP2040-Tiny                              |
 |       ^                                                                     |
 |       +-- EN <- STORAGE_SW <- SYS ; EN pulldown 100 kOhm                   |
 |                                                                             |
 +--> MAIN_PWR : JMTQ55P02A --> Q6A J19                                       |
 |       commande AO3400A + maintien RC ~1 s                                  |
 |                                                                             |
 +--> MODEM_PWR : JMTQ55P02A --> EC25 carrier BAT                             |
 |       commande AO3400A + maintien RC ~1 s                                  |
 |                                                                             |
 +--> ANNEXE1 : P-MOS Basic, OFF par défaut                                    |
 +--> ANNEXE2 : P-MOS Basic, OFF par défaut                                    |

Q6A USB2 HOST #1 ---- VBUS/D+/D-/GND ----> EC25 VBUS/DP/DN/GND
Q6A USB2 HOST #2 -------------------------> caméra USB future
Q6A USB2 HOST #3 -------------------------> réserve / futur USB-A amovible

RP2040-Tiny
  <-> BQ25628E I2C + INT
  <-> Q6A SBS/I2C
   -> Q6A PWR_ON_KEY via S8050
   -> Q6A SLEEP_REQ
   <- Q6A HEARTBEAT
  <-> EC25 UART/DTR/RI via SN74AVC4T245
   -> EC25 PWRKEY via S8050
   -> MAIN_PWR / MODEM_PWR / ANNEXE1 / ANNEXE2
   <- bouton Power utilisateur
   <- CC_SENSE analogique fusionné
```

Le PCB **ne transporte plus l'USB Q6A <-> EC25**. Cette liaison est un câble/faisceau court direct.

---

# 1. Batterie et domaine SYS

## 1.1 Batterie

```text
Type            : Li-ion/LiPo 1S classique
Tension max     : 4.20 V ; pas de cellule HV 4.35/4.40 V
Capacité cible  : ~5000 mAh
Protection      : pack protégé / PCM obligatoire
NTC             : 10 kOhm type Semitec 103AT-2 ou strictement compatible
Courant cible   : pack/PCM/câblage capable d'encaisser les pics combinés Q6A + LTE
```

Objectif raisonnable de sélection pack : **au moins ~5 A continu avec marge de pic**, à confirmer sur la fiche réelle de la cellule/PCM. Le seuil OCP du BATFET du BQ n'est pas à interpréter comme un courant continu garanti.

Le chemin SYS doit être dimensionné pour les pics simultanés : Q6A, EC25, MCU et annexes.

## 1.2 Fusible batterie

```text
F_BAT  : footprint optionnel DNP
SJ_BAT : jumper cuivre fermé par défaut
```

Le pack protégé reste obligatoire même si un fusible est ajouté plus tard.

---

# 2. BQ25628E — chargeur / power-path

## 2.1 Référence

```text
U_CHG       : BQ25628ERYKR
JLC/LCSC    : C18221178
Boîtier     : WQFN-18 2.5 x 3 mm
Classe      : Extended / Economic
Batterie    : 1S
Charge max  : 2 A
Power-path  : NVDC
ADC/I2C     : oui
```

Le BQ fournit la télémétrie VBUS/VBAT/VSYS/IBUS/IBAT/TS et les états charge/fault. Il n'est pas considéré comme un fuel-gauge précis à lui seul.

## 2.2 Pins

```text
CE   -> GND
SDA  -> GP4 + pull-up 10 kOhm vers 3V3_MCU
SCL  -> GP5 + pull-up 10 kOhm vers 3V3_MCU
INT  -> GP6 + pull-up 10 kOhm vers 3V3_MCU
PG   -> testpoint
STAT -> NC
QON  -> testpoint / pull-up interne conservé
```

La charge doit rester sûre même si le MCU est arrêté. Le MCU affine les paramètres par I2C lorsqu'il fonctionne.

## 2.3 ILIM au boot

Retenir :

```text
RILIM = 5.6 kOhm / C23189 Basic / 1 %
```

Cible : ~0.45 A typique et rester sous ~0.5 A malgré la dispersion de KILIM. Le firmware conserve `EN_EXTILIM` tant que l'annonce Type-C n'est pas reconnue.

Prévoir si le placement est gratuit deux pads DNP pour le réseau RC optionnel recommandé par TI en cas de besoin ultérieur sur ILIM.

## 2.4 Politique firmware BQ de départ

```text
VREG nominal        : 4.15 V
VSYSMIN             : 3.84 V
source inconnue     : IINDPM conservateur ~0.44 A
Rp 1.5 A reconnu    : IINDPM avec marge, ~1.35 A max de départ
Rp 3.0 A reconnu    : IINDPM avec marge, <= capacité réelle entrée/BQ
ICHG                : <=2 A et abaissé selon thermique / charge système
```

`VREG=4.15 V` est une politique de départ volontairement prudente : elle ajoute de la marge au EC25 alimenté depuis SYS et réduit légèrement le stress cellule. Elle pourra être ajustée après mesures, sans changement PCB.

## 2.5 Tension MODEM_PWR

Quectel spécifie le domaine EC25 à **3.3–4.3 V, 3.8 V typique**, avec alimentation capable de pics proches de 2 A. Le BQ peut réguler SYS typiquement environ 50 mV au-dessus de BAT lorsque BAT est au-dessus de VSYSMIN.

Firmware de départ :

```text
MODEM_PWR autorisé si environ 3.35 V <= VSYS <= 4.25 V
```

Règle matérielle absolue à vérifier au banc :

```text
EC25 carrier BAT doit rester < 4.30 V dans tous les états,
y compris batterie pleine, USB branché et transitoires.
```

En dessous de ~3.35 V, arrêter proprement le modem avant MODEM_PWR OFF.

## 2.6 NTC / TS

```text
RT1 = 5.1 kOhm / C25905 Basic
RT2 = 30 kOhm  / C22984 Basic
NTC = 10 kOhm 103AT-2 compatible, hors PCB
```

Validation thermique obligatoire avant d'autoriser 2 A de charge.

## 2.7 Passifs BQ

```text
L_CHG    : XRIM252012S1R0MBCA / C22471110
           1 uH, 4 A rated, 5.6 A Isat, ~35 mOhm

CVBUS    : 1 x 1 uF / C52923 Basic
CVBUS_HF : 1 x 100 nF / C1525 Basic
CPMID    : 2 x 10 uF / C15850 Basic
CPMID_HF : 1 x 100 nF / C1525 Basic
CSYS     : 3 x 10 uF / C15850 Basic
CBAT     : 1 x 1 uF / C52923 Basic
CREGN    : 1 x 4.7 uF / C1779 Basic
CBTST    : 1 x 47 nF 50 V / C1622 Basic, BTST-SW
```

## 2.8 Layout BQ

Le bloc doit être rerouté selon la recommandation TI, même si l'ancien PCB passait le DRC :

```text
- PMID caps collés au pin PMID/GND ;
- boucle PMID -> switch -> inductance -> SYS -> caps -> GND minimale ;
- SYS caps collés au pin SYS ;
- CVBUS/CBAT/REGN au plus près ;
- SW le plus petit possible ;
- vias GND/thermiques sous et autour du BQ ;
- SW éloigné de TS, I2C, CC et ADC ;
- BAT/SYS/MAIN/MODEM très larges ;
- vias multiples si changement de couche sur puissance.
```

---

# 3. USB-C extérieur — charge + device uniquement

## 3.1 Fonction

```text
Charge 5 V
ADB
EDL
MTP / USB device
```

**Aucun host USB-C, aucun DRP, aucun contrôleur CC, aucun PD V1.**

Le host utilisateur futur passe par le troisième port USB2 host libre du Q6A avec un connecteur/câble amovible dédié.

## 3.2 Connecteur et data

```text
TYPE-C-31-M-12 / C165948
A6+B6 -> D+ -> ESD -> Q6A OTG D+
A7+B7 -> D- -> ESD -> Q6A OTG D-
VBUS  -> TVS -> BQ VBUS
```

Routage D+/D- : vraie paire USB2 ~90 Ohm différentiel selon le stack-up JLC réel, même couche, plan GND continu, aucun stub, peu de vias.

## 3.3 VBUS Q6A

```text
USB-C VBUS -> anode B5819W SL / C8598 Basic
cathode    -> Q6A OTG VBUS
```

Cette Schottky évite qu'un éventuel 5 V Q6A forcé en host retourne vers USB-C/BQ.

**Gate obligatoire :** avec MAIN_PWR OFF et USB-C branché, vérifier que cette ligne VBUS et D+/D- ne back-powerent pas significativement le Q6A.

## 3.4 CC1/CC2 et ADC fusionné

```text
CC1 -> 5.1 kOhm -> GND / C25905 Basic
CC2 -> 5.1 kOhm -> GND / C25905 Basic

CC1 -- 470 kOhm / C23178 --+
                            +--> CC_SENSE -> GP26 / ADC0
CC2 -- 470 kOhm / C23178 --+
                            |
                         100 nF / C1525
                            |
                           GND
```

Une seule CC est active selon l'orientation. Le point ADC voit environ la moitié de la tension CC active. Firmware : attendre la stabilisation, moyenner, et considérer toute mesure ambiguë comme source Default.

## 3.5 Protection

```text
U_ESD  : SRV05-4 / C558418 — D+, D-, CC1, CC2
D_VBUS : SMF5.0A / C193402 — VBUS
```

Pas de common-mode choke USB2 en V1.

---

# 4. RP2040-Tiny et Storage OFF

## 4.1 Module

Waveshare RP2040-Tiny soudé manuellement sur la face BOTTOM, sans headers.

Contraintes :

```text
- footprint vérifié sur module réel ;
- impression 1:1 ;
- orientation contrôlée ;
- congés de soudure inspectables ;
- keepout correct sous module ;
- FPC accessible après assemblage ;
- aucun via/testpoint susceptible de toucher le dessous du module.
```

Le FPC du RP2040-Tiny reste le chemin de récupération USB/BOOTSEL/RUN ; aucun SWD supplémentaire n'est nécessaire en V1.

## 4.2 Alimentation MCU

```text
SYS -> TPS610995DRVR / C2071098 -> MCU_VSYS -> VSYS RP2040-Tiny
L_MCU : MAKK2016T2R2M / C92923 / 2.2 uH
CIN   : 10 uF / C15850
COUT  : 2 x 10 uF / C15850
```

Le rail est nommé `MCU_VSYS`, pas `3V6_MCU`, car le TPS610995 peut passer en régime de pass-through lorsque VIN est suffisamment élevé.

## 4.3 Storage switch

```text
SYS -> STORAGE_SW -> EN TPS610995
EN -> 100 kOhm -> GND
```

```text
switch fermé : MCU alimenté
switch ouvert: hard-off MCU / stockage
```

Le switch Storage ne remplace pas le shutdown normal. Si ouvert brutalement pendant RUN, le RC MAIN/MODEM maintient environ une seconde puis les rails s'éteignent.

---

# 5. MAIN_PWR et MODEM_PWR — default OFF + maintien reset MCU

## 5.1 P-MOS puissance

```text
Q_MAIN_P  : JMTQ55P02A / C2890429
Q_MODEM_P : JMTQ55P02A / C2890429
P-MOS 20 V, ~12 mOhm @ |VGS|=2.5 V
```

Les deux rails restent **OFF par défaut** si le MCU n'existe pas ou reste arrêté.

## 5.2 Driver validé

Pour chaque rail :

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
AO3400A : C20917 Basic
10 kOhm : C25744 Basic
100 kOhm: C25741 Basic
1 MOhm  : C22935 Basic
1 uF    : C15849 Basic
```

Constante RC nominale : ~1 s. Le délai réel dépend du seuil du AO3400A et doit être mesuré ; cible pratique ~0.7–1.5 s.

Fonctionnement :

```text
GPIO HIGH     -> rail ON
GPIO LOW      -> arrêt volontaire rapide
GPIO Hi-Z     -> C_HOLD maintient momentanément ON
MCU reboot    -> le firmware réactive immédiatement le GPIO avant décharge complète
MCU mort long -> RC expire -> rail OFF
```

Le condensateur est placé sur la **grille du petit NMOS de commande**, jamais sur la grille du P-MOS de puissance : le P-MOS commute rapidement et n'est pas maintenu longtemps dans sa zone linéaire.

## 5.3 Boot firmware RP2040

GP10 et GP29 doivent être lus immédiatement au reset avant reconfiguration : si le niveau RC est encore haut, les passer sans délai en sortie HIGH. Ensuite heartbeat/états BQ/EC25 resynchronisent la machine d'états.

---

# 6. ANNEXE1 / ANNEXE2

Branches simples, **OFF par défaut**, sans RC de maintien.

Référence de puissance retenue pour économiser les P-MOS Extended :

```text
AO3401A / C15127 Basic
limite cible : ~1 A continu par annexe
```

Commande par petit transistor/NMOS logique avec pull-down ; aucun maintien pendant reset MCU.

Une annexe peut alimenter l'ampli stéréo et être coupée pendant un appel.

---

# 7. Q6A <-> MCU

## 7.1 Affectation matérielle

```text
Q6A J20 pin 36 / GPIO59 -> Q6A_HEARTBEAT -> GP27
Q6A J20 pin 37 / GPIO58 <- Q6A_SLEEP_REQ <- GP9
Q6A J20 pin 3  / GPIO24 / I2C6_SDA <-> GP14 SBS_SDA
Q6A J20 pin 5  / GPIO25 / I2C6_SCL <-> GP15 SBS_SCL
Q6A J20 pin 1 ou 17 / 3V3 -> Q6A_3V3_REF
Q6A GND -> GND
Q6A PWR_ON_KEY <- S8050 <- GP8
```

`GPIO59 WAKE` est abandonné. Le réveil utilise le **vrai PWR_ON_KEY du PMIC**, qui sert aussi au cold boot.

## 7.2 Q6A PWR_ON_KEY

```text
GP8 -> 10 kOhm -> base S8050 / C2146 Basic
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Usage :

```text
- cold boot après activation MAIN_PWR ;
- wake depuis deep ;
- événement Power Android ;
- appui long de récupération si nécessaire.
```

Pulse normal initial à tester autour de 100–300 ms.

## 7.3 SLEEP_REQ

La ligne reste distincte de PWRKEY car notre suspend normal doit exécuter une séquence propre : modem USB, DTR/QSCLK, OTG externe `none`, Wi-Fi/wake sources, puis `mem_sleep=deep`.

Câblage de départ :

```text
Q6A_3V3 -> 100 kOhm -> SLEEP_REQ
SLEEP_REQ -> 10 kOhm série -> GP9
```

Firmware MCU : actif = sortie LOW ; repos = entrée/Hi-Z.

## 7.4 HEARTBEAT

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Le heartbeat est un **statut logiciel**, pas une sécurité combinatoire directe. Il peut coder des états par fréquence/pattern : RUN, transition suspend/shutdown, etc. En deep sleep il peut volontairement s'arrêter.

Règle : perte heartbeat inattendue **ne coupe jamais immédiatement MAIN_PWR**. Le MCU tente d'abord wake/récupération, puis power-cycle après timeout explicite.

## 7.5 SBS / batterie virtuelle

```text
GP14 <-> Q6A I2C6_SDA
GP15 <-> Q6A I2C6_SCL
```

Prévoir sur la carte Power :

```text
2 x 4.7 kOhm vers Q6A_3V3, DNP par défaut
```

On ne les peuple que si le Q6A n'a pas déjà les pull-up nécessaires. Ne jamais tirer SDA/SCL vers 3V3_MCU, afin d'éviter le back-power du Q6A lorsqu'il est coupé.

L'interface SBS est utile pour exposer batterie/courant/température à Android mais n'est pas requise pour booter.

---

# 8. EC25 — alimentation, USB, supervision

## 8.1 Alimentation principale

```text
SYS -> MODEM_PWR -> EC25 carrier BAT
GND -> EC25 carrier GND
```

Le carrier `BAT` est l'alimentation principale du modem. `USB_VBUS` sert au domaine USB/détection et ne doit pas porter les pics LTE principaux.

## 8.2 USB direct depuis Q6A

```text
Q6A USB2 host VBUS -> EC25 VBUS
Q6A USB2 host D+   -> EC25 DP
Q6A USB2 host D-   -> EC25 DN
Q6A GND            -> EC25 GND
```

Pas de switch VBUS EC25 en V1. La liaison directe doit être courte ; D+/D- torsadés ensemble avec masse proche.

Gates :

```text
- vérifier que le host Q6A suspend réellement l'USB en deep ;
- vérifier que VBUS présent avec MODEM_PWR OFF ne back-power pas significativement l'EC25 ;
- si le sommeil échoue, le câble direct pourra être modifié plus tard pour interrompre VBUS.
```

## 8.3 UART / DTR / RI

```text
SN74AVC4T245DR / C22495
VCCA = EC25 VIO (~1.8 V)
VCCB = 3V3_MCU

EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

Le composant est intégralement occupé par ces 4 signaux.

Prévoir :

```text
VIO -> 100 kOhm -> GND
EC25 DTR -> 100 kOhm -> VIO
EC25 RXD -> 100 kOhm -> VIO
100 nF sur VCCA
100 nF sur VCCB
```

Mesurer VIO/TXD/RI sur le carrier réel avant connexion définitive.

## 8.4 EC25 PWRKEY

```text
GP13 -> 10 kOhm -> base S8050 / C2146
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> EC25 PWRKEY
```

Ordre de récupération : AT propre -> PWRKEY -> MODEM_PWR power-cycle.

## 8.5 EC25 RESET_N / GP28

`RESET_N` n'occupe plus un GPIO normal. GP28 devient la réserve de dépannage V1.

```text
GP28 -> TP_GP28_3V3
```

Conserver un étage DNP optionnel :

```text
GP28 -> 10 kOhm DNP -> base S8050 DNP
base -> 100 kOhm DNP -> GND
émetteur -> GND
collecteur -> TP_EC25_RST_OD -> strap/pad DNP -> EC25 RESET_N
```

On obtient :

```text
- un vrai GPIO 3.3 V sur TP_GP28_3V3 ;
- après l'étage, une sortie open-drain utilisable dans le domaine 1.8 V / RESET_N.
```

Ce n'est **pas** un GPIO push-pull 1.8 V ; ajouter un level-shifter 1 bit uniquement pour cette réserve n'est pas justifié en V1.

---

# 9. Pinout RP2040-Tiny final

```text
GP0   -> EC25 RXD via SN74AVC4T245       (UART TX MCU)
GP1   <- EC25 TXD via SN74AVC4T245       (UART RX MCU)
GP2   -> EC25 DTR via SN74AVC4T245
GP3   <- EC25 RI via SN74AVC4T245
GP4   <-> BQ SDA
GP5   ->  BQ SCL
GP6   <-  BQ INT
GP7   <-  PWR_BUTTON utilisateur, actif bas
GP8   ->  Q6A PWR_ON_KEY via S8050
GP9   ->  Q6A SLEEP_REQ, actif bas / Hi-Z au repos
GP10  ->  MAIN_PWR_EN avec RC hold
GP11  ->  ANNEXE1_EN
GP12  ->  ANNEXE2_EN
GP13  ->  EC25 PWRKEY via S8050
GP14  <-> Q6A SBS_SDA
GP15  <-> Q6A SBS_SCL
GP26  <-  USB_CC_SENSE / ADC0, CC1+CC2 fusionnés
GP27  <-  Q6A HEARTBEAT / GPIO59
GP28  <-> RESERVE / TP 3.3 V / option RESET_N open-drain DNP
GP29  ->  MODEM_PWR_EN avec RC hold
```

Toutes les GPIO exposées sont affectées, mais **GP28 reste volontairement une réserve**.

Bouton Power utilisateur : contact NO entre GP7 et GND, pull-up 10 kOhm vers 3V3_MCU.

---

# 10. Machine d'états Power

## 10.1 Cold boot

```text
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

## 10.2 Reset MCU pendant RUN

```text
RC MAIN/MODEM maintient les deux rails ~1 s
-> RP2040 reboot
-> lit immédiatement GP10/GP29
-> les force HIGH si encore maintenus
-> aucune coupure Q6A/modem
-> resynchronisation via HEARTBEAT/BQ/UART ensuite
```

## 10.3 Suspend normal

```text
bouton utilisateur -> MCU
-> SLEEP_REQ
-> Q6A prépare le suspend :
   EC25 QSCLK/DTR + USB host suspend
   USB OTG externe -> none
   Wi-Fi/autres wake sources traités
   état heartbeat adapté
-> mem_sleep=deep
-> MAIN_PWR reste ON
-> MODEM_PWR reste ON si modem doit rester joignable
```

Réveil : pulse `Q6A PWR_ON_KEY`.

## 10.4 Shutdown normal

```text
1. shutdown Android propre ;
2. arrêt EC25 propre ;
3. timeout/confirmation ;
4. MODEM_PWR OFF ;
5. MAIN_PWR OFF ;
6. MCU reste ON tant que STORAGE_SW fermé.
```

## 10.5 Storage OFF

Après shutdown propre uniquement : ouvrir `STORAGE_SW`. MCU s'éteint ; MAIN/MODEM sont déjà OFF. Si le switch est ouvert accidentellement en RUN, le RC ne garantit qu'environ une seconde avant coupure.

---

# 11. SOC / batterie Android

Le MCU exploite la télémétrie BQ et une courbe cellule pour produire une estimation V1 :

```text
VBAT / VSYS / IBAT / IBUS / TS
état charge/faults
courbe OCV/SOC
intégration logicielle lorsque pertinente
```

Cette estimation est exposable à Android via SBS mais **n'est pas un fuel-gauge de précision**.

`MAX17048` n'est pas requis pour la baseline V1. Si un footprint DNP existe déjà et ne coûte pratiquement aucune surface, il peut être conservé comme fallback expérimental ; il ne doit pas être une dépendance de fonctionnement.

---

# 12. Connectique / testpoints

## 12.1 Puissance

```text
J_BAT : Molex Micro-Fit 436500300 / C503478 — BAT+ / GND / NTC
J_Q6A : Molex Micro-Fit 436500200 / C192562 — MAIN_PWR+ / GND
J_A1  : Molex Micro-Fit 436500200 / C192562 — ANNEXE1+ / GND
J_A2  : Molex Micro-Fit 436500200 / C192562 — ANNEXE2+ / GND
```

MODEM_PWR : gros pads THT BAT/GND vers carrier EC25 afin d'éviter un connecteur supplémentaire.

STORAGE_SW : deux pads THT accessibles.

## 12.2 Testpoints obligatoires

```text
VBUS_USB_C / BQ_VBUS / BAT / SYS
MCU_VSYS / 3V3_MCU / TPS610995_EN / GND
BQ_SDA / BQ_SCL / BQ_INT / BQ_TS / BQ_ILIM
MAIN_PWR_GATE / MAIN_PWR_OUT
MODEM_PWR_GATE / MODEM_PWR_OUT
ANNEXE1_OUT / ANNEXE2_OUT
CC1 / CC2 / CC_SENSE
Q6A_3V3_REF / Q6A_PWRKEY / Q6A_SLEEP_REQ / Q6A_HEARTBEAT
SBS_SDA / SBS_SCL
EC25_VIO / TXD / RXD / DTR / RI / PWRKEY
TP_GP28_3V3 / TP_EC25_RST_OD
```

---

# 13. PCB / mécanique

```text
PCB : 4 couches
max strict : 70 x 25 mm
RP2040-Tiny : BOTTOM
```

Organisation :

```text
L1 : composants + USB2 + boucles puissance locales
L2 : GND continu
L3 : puissance + signaux lents
L4 : signaux + GND
```

Priorités de placement : BQ et ses boucles d'abord, puis USB-C/protection, puis rails de puissance ; le RP2040 reste mécaniquement accessible sur BOTTOM.

---

# 14. BOM — choix principaux

## Extended conservés

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

Les quantités exactes sont à régénérer depuis le schéma V0.5 ; ne pas recopier la BOM chiffrée du 30/09.

---

# 15. Fonctions explicitement supprimées

```text
USB-C host / DRP / contrôleur CC   -> supprimé V1
USB-PD / 9 V                       -> supprimé V1
Q6A GPIO59 WAKE                    -> remplacé par PWR_ON_KEY ; GPIO59 = HEARTBEAT
CC1 et CC2 sur deux ADC            -> fusionnés sur GP26
EC25 puissance par VBUS Q6A        -> supprimée ; BAT depuis MODEM_PWR
USB EC25 à travers PCB Power       -> supprimé ; câble direct
switch USB_VBUS EC25               -> non monté V1
EC25 RESET dédié                   -> supprimé ; option DNP GP28
DISPLAY_PWR                        -> absent de cette carte
codec audio Minimal EC25           -> supprimé
jack utilisateur                   -> supprimé
mux audio analogique               -> supprimé
CMC USB2                           -> non monté V1
```

---

# 16. Gates BLOQUANTES avant fabrication

```text
[ ] Q6A J19 : R7 retiré ; R24 2 mOhm ; R190 100 k ; R191 10 k ; FB4 DNP ; R185..R189 confirmés DNP.
[ ] Q6A J19 au banc : 4.2 / 3.8 / 3.4 / 3.0 V, boot + idle + CPU + vrai deep.
[ ] Q6A PWR_ON_KEY : cold boot et wake deep validés avec l'architecture J19.
[ ] GPIO58 SLEEP_REQ -> procédure deep ; GPIO59 -> heartbeat logiciel.
[ ] EC25 VIO/TXD/RI mesurés sur carrier réel.
[ ] EC25 BAT depuis SYS/MODEM_PWR : jamais >=4.30 V ; observer transitoires à l'oscilloscope.
[ ] EC25 sous pics LTE : VSYS/BAT stables ; pas de reset Q6A/modem.
[ ] EC25 USB host Q6A : vrai USB suspend en deep avec VBUS connecté.
[ ] MODEM_PWR OFF + Q6A VBUS présent : absence de back-power EC25 problématique.
[ ] USB-C branché + MAIN_PWR OFF : absence de back-power Q6A problématique par OTG VBUS/D+/D-.
[ ] Pack/PCM/câblage : courant admissible documenté avec marge suffisante.
[ ] TPS610995 : shutdown/isolation/storage current et back-power validés.
[ ] SBS : vérifier pull-up existants Q6A avant peuplement des 4.7 k DNP.
[ ] Footprint RP2040-Tiny + impression 1:1 + FPC accessible.
[ ] Layout BQ comparé visuellement au layout TI avant DRC final.
[ ] USB-C D+/D- recalculé avec stack-up JLC choisi.
[ ] Analyse back-power complète : Q6A, EC25, BQ, SBS, heartbeat, CC, USB, RP2040.
[ ] ERC/DRC propres ; footprints, polarités, BOM/PnP/Gerbers revus manuellement.
[ ] PCB <=70 x 25 mm sans collision TOP/BOTTOM.
```

Fonctions radio Android avancées (data complète, SMS, IMS/VoLTE, audio final) ne bloquent pas la carte Power si les interfaces électriques sont validées.

---

# 17. Tests après fabrication

```text
[ ] USB-C 5 V -> BQ -> SYS sans batterie
[ ] batterie seule -> SYS
[ ] plug/unplug USB sans reboot
[ ] ILIM ~0.45 A au boot
[ ] CC_SENSE Default / 1.5 A / 3 A
[ ] IINDPM / EN_EXTILIM corrects
[ ] charge jusqu'à 2 A caractérisée thermiquement
[ ] NTC / télémétrie BQ
[ ] RP stable sur plage SYS
[ ] Storage OFF courant résiduel
[ ] MAIN/MODEM OFF par défaut
[ ] reset MCU court : MAIN/MODEM ne chutent pas
[ ] arrêt volontaire : coupure rapide malgré C_HOLD
[ ] Q6A PWRKEY cold boot / wake / long press
[ ] SLEEP_REQ + heartbeat
[ ] SBS sans back-power
[ ] USB-C ADB/EDL/MTP
[ ] aucun retour VBUS Q6A vers USB-C
[ ] EC25 BAT dans plage sûre
[ ] EC25 USB direct Q6A stable
[ ] EC25 UART/DTR/RI
[ ] EC25 sleep avec USB suspend
[ ] EC25 PWRKEY / shutdown / recovery
[ ] consommations deep et storage-off
```

---

# 18. Sources primaires de conception

- Texas Instruments — BQ25628E datasheet Rev. C / NVDC power-path.
- Texas Instruments — TPS61099x documentation.
- Texas Instruments — SN74AVC4T245 documentation.
- Quectel — EC25 Series Hardware Design V2.4.
- Radxa — Dragon Q6A V1.21 schematic + GPIO/USB documentation.
- Waveshare — RP2040-Tiny schematic/mécanique.
- JLCPCB/LCSC — bibliothèque de composants ; classifications à revalider avant commande.

---

# 19. Gel

L'architecture électrique est **figée pour produire le schéma V0.5**. Les points restants sont principalement des mesures de validation physique. Toute modification qui réintroduit USB-C host/DRP, supprime MODEM_PWR, remplace PWR_ON_KEY par GPIO59 WAKE ou sépare de nouveau CC1/CC2 doit être considérée comme une révision d'architecture et non une simple correction de schéma.
