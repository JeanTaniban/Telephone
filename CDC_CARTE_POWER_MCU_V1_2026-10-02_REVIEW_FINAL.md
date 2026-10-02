# CDC — Carte Power / MCU V1 du Maker Phone Q6A — revue finale

**Date :** 2026-10-02  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **CANDIDAT FIGÉ APRÈS AUDIT ÉLECTRIQUE — en attente de validation utilisateur avant promotion comme CDC courant**  
**Fabrication cible :** JLCPCB Economic PCBA  
**Priorité composants :** Basic > Promotional > Extended. Les références Extended conservées le sont pour une raison électrique, mécanique ou de disponibilité vérifiée.

Ce document remplace, pour la revue utilisateur, les choix devenus obsolètes dans `CDC_CARTE_POWER_MCU_V1_2026-09-30.md` : alimentation modem par VBUS Q6A, absence de MODEM_PWR, GPIO59 utilisé comme WAKE, CC1/CC2 sur deux ADC, EC25_RST dédié, commande MAIN_PWR sans maintien RC.

---

# 0. Verdict de l'audit

L'architecture est cohérente et peut être figée pour le schéma V0.5 sous réserve des gates de validation physique listées en fin de document.

Corrections importantes apportées pendant l'audit :

```text
- EC25 alimenté depuis SYS par MODEM_PWR séparé ; USB Q6A ne fournit plus sa puissance principale.
- MAIN_PWR et MODEM_PWR = OFF par défaut mais maintien ~1 s lors d'un reset MCU.
- commande de ces deux rails par AO3400A Basic + RC sur la grille du NMOS, jamais par RC sur la grille du P-MOS de puissance.
- réveil Q6A par son vrai PWR_ON_KEY ; GPIO59 devient HEARTBEAT.
- GPIO58 reste SLEEP_REQ pour lancer la procédure de suspend propre.
- CC1/CC2 fusionnés vers un seul ADC GP26.
- GP27 récupéré pour HEARTBEAT ; GP28 devient GPIO de service ; GP29 commande MODEM_PWR.
- EC25 RESET_N n'est plus une fonction normale ; uniquement option DNP via open-collector.
- USB-C externe définitivement sink/device : charge + ADB/EDL/MTP, aucun mode host Type-C.
- RILIM BQ passe de 5.1 kOhm à 5.6 kOhm pour rester sous ~500 mA même avec la dispersion KILIM.
- VREG firmware nominal = 4.15 V afin de laisser de la marge à l'EC25 alimenté depuis SYS.
- VSYSMIN firmware = 3.84 V, valeur maximale du BQ25628E.
- MODEM_PWR autorisé uniquement si VSYS est dans une fenêtre sûre ~3.35 V à 4.20 V.
- TPS610995 conservé ; son rail est nommé MCU_VSYS car le composant peut passer VIN lorsque VIN > VOUT + ~0.5 V.
- condensateur BAT du BQ ramené à la valeur de référence 1 uF ; marge renforcée conservée sur PMID/SYS.
- ANNEXE1/2 passent sur AO3401A Basic, acceptable pour une limite de 1 A continu.
- MAX17048 supprimé de la V1 finale ; SOC estimé par MCU à partir du BQ et de la courbe cellule, sans prétendre à la précision d'un fuel-gauge dédié.
```

---

# 1. Architecture système figée

```text
USB-C extérieur 5 V / USB2 DEVICE uniquement
        |
        +--> VBUS --> TVS --> BQ25628E --> SYS --------------------------+
        |                         |                                       |
        |                         +--> BAT --> batterie 1S protégée + NTC |
        |                                                                 |
        +--> D+/D- -----------------------------> Q6A USB OTG DEVICE      |
        +--> VBUS -- Schottky --> Q6A OTG VBUS detect                    |
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
 +--> ANNEXE1 : AO3401A --> sortie 1, max cible 1 A continu                    |
 +--> ANNEXE2 : AO3401A --> sortie 2, max cible 1 A continu                    |

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

Le PCB ne transporte plus l'USB Q6A <-> EC25 : liaison directe par fils/câble court.

---

# 2. Batterie / contraintes de courant

## 2.1 Batterie

```text
Type            : Li-ion/LiPo 1S classique 4.20 V max
Capacité cible  : ~5000 mAh
NTC             : 10 kOhm type Semitec 103AT-2 ou compatible strict
Protection      : PCM/protection pack obligatoire
Courant pack    : cible >= 6 A continu et >= 10 A court pic si la fiche du pack le permet
Câblage         : faible résistance, adapté aux pics Q6A + LTE
```

Ne pas utiliser une cellule HV 4.35/4.40 V avec cette architecture : l'EC25 est directement dérivé de SYS et son domaine d'alimentation ne doit pas dépasser 4.3 V.

Le BQ25628E accepte 6 A RMS de décharge batterie et 10 A de pic court au niveau de son BATFET ; la cellule, son PCM, le connecteur et les pistes doivent être cohérents avec le courant système réel.

## 2.2 Fusible

Conserver :

```text
F_BAT  : footprint optionnel DNP
SJ_BAT : solder-jumper cuivre fermé par défaut
```

Le pack protégé reste obligatoire même si un fusible est ajouté ultérieurement.

---

# 3. Chargeur / power-path BQ25628E

## 3.1 Composant

```text
U_CHG       : BQ25628ERYKR
JLC/LCSC    : C18221178
Boîtier     : WQFN-18 2.5 x 3 mm
Classe      : Extended / Economic
Batterie    : 1S
Charge max  : 2 A
Power-path  : NVDC intégré
ADC/I2C     : oui
```

Aucune substitution Basic retenue : le bloc chargeur/power-path/ADC est central et la référence choisie reste pertinente.

## 3.2 Configuration matérielle des pins

```text
CE   -> GND
SDA  -> GP4 + pull-up 10 kOhm vers 3V3_MCU
SCL  -> GP5 + pull-up 10 kOhm vers 3V3_MCU
INT  -> GP6 + pull-up 10 kOhm vers 3V3_MCU
PG   -> testpoint uniquement
STAT -> NC
QON  -> testpoint, pull-up interne conservé
```

`CE=LOW` autorise la charge autonome. Avec MCU éteint et USB présent, le BQ peut donc charger de manière autonome avec ses paramètres POR. Le MCU reprogramme les paramètres dès son démarrage.

## 3.3 Limite d'entrée au boot — correction importante

```text
RILIM = 5.6 kOhm / C23189 Basic / 1 %
```

Avec KILIM typique ~2500 AOhm :

```text
IILIM typique ~= 2500 / 5600 ~= 446 mA
```

Avec KILIM haut ~2750 AOhm :

```text
IILIM ~= 491 mA
```

Cette valeur remplace 5.1 kOhm : 5.1 kOhm pouvait dépasser 500 mA avec la dispersion KILIM.

Le RC supplémentaire recommandé par TI pour ILIM < 400 mA n'est pas monté : la consigne nominale reste >400 mA. Prévoir toutefois deux petits pads DNP si le placement ne coûte rien pour permettre ultérieurement la branche RC TI 1.2 kOhm + 330 nF en parallèle de RILIM.

## 3.4 Politique firmware BQ figée

Au démarrage MCU :

```text
1. vérifier l'identifiant BQ et les faults ;
2. VREG = 4.15 V en fonctionnement normal ;
3. VSYSMIN = 3.84 V ;
4. conserver EN_EXTILIM actif tant que le courant Type-C n'est pas identifié ;
5. mesurer CC_SENSE après stabilisation ;
6. programmer IINDPM avec marge ;
7. seulement pour une source 1.5 A / 3 A reconnue, désactiver EN_EXTILIM si nécessaire ;
8. laisser DPM réduire automatiquement la charge lorsque le système consomme beaucoup.
```

Politique de départ :

```text
CC inconnu/default : IINDPM <= ~0.44 A ; charge conservatrice
Rp 1.5 A reconnu   : IINDPM <= ~1.35 A ; ICHG <= ~1.0 A
Rp 3.0 A reconnu   : IINDPM <= ~2.7 A ; ICHG <= 2.0 A
```

Ces valeurs sont des plafonds firmware. Elles peuvent être abaissées thermiquement ou lorsque Q6A/EC25 tirent du courant.

`VREG=4.15 V` est volontaire : gain de marge pour l'EC25 et meilleur vieillissement cellule. Si le téléphone est en STORAGE_OFF avec USB branché, le BQ peut revenir au POR 4.20 V puisque le MCU est éteint ; MODEM_PWR reste alors OFF par défaut. Après réveil MCU, le modem n'est activé qu'après retour de VSYS dans sa fenêtre sûre.

## 3.5 Fenêtre MODEM_PWR obligatoire

L'EC25 accepte environ 3.3 à 4.3 V. Le BQ régule typiquement SYS ~50 mV au-dessus de BAT lorsque BAT est au-dessus de VSYSMIN et la charge terminée/désactivée.

Le firmware ne doit donc autoriser MODEM_PWR que si :

```text
3.35 V <= VSYS_ADC <= 4.20 V
```

Au-dessus de 4.20 V : attendre que Q6A / le système fasse descendre la batterie/SYS.  
En dessous de 3.35 V : arrêter proprement le modem avant de couper MODEM_PWR.

La mesure réelle de SYS à batterie pleine reste une gate avant fabrication définitive. Le seuil logiciel peut ensuite être affiné, mais il ne doit jamais autoriser une tension approchant 4.30 V sur EC25 BAT.

## 3.6 NTC / TS

Conserver le réseau TI adapté au 103AT :

```text
RT1 = 5.1 kOhm / C25905 Basic
RT2 = 30 kOhm  / C22984 Basic
NTC = 10 kOhm 103AT-2 compatible, hors PCB
```

L'écart vis-à-vis des valeurs nominales TI plus précises est faible et accepté pour privilégier Basic. Les seuils thermiques réels doivent être testés avant autorisation charge 2 A.

## 3.7 Passifs BQ figés

```text
L_CHG    : XRIM252012S1R0MBCA / C22471110
           1 uH, 4 A rated, 5.6 A Isat, ~35 mOhm
           Extended / Economic

CVBUS    : 1 x 1 uF / C52923 Basic
CVBUS_HF : 1 x 100 nF / C1525 Basic
CPMID    : 2 x 10 uF / C15850 Basic
CPMID_HF : 1 x 100 nF / C1525 Basic
CSYS     : 3 x 10 uF / C15850 Basic
CBAT     : 1 x 1 uF / C52923 Basic
CREGN    : 1 x 4.7 uF / C1779 Basic
CBTST    : 1 x 47 nF 50 V / C1622 Basic, entre BTST et SW
```

`CBAT` 1 uF suit la valeur utilisée par TI ; les anciens 2 x 10 uF côté BAT sont supprimés comme inutiles. La marge de capacité reste volontairement élevée sur PMID et SYS.

## 3.8 Layout BQ — obligatoire

Le layout doit être refait autour du BQ selon le sens TI, pas seulement réutilisé parce que l'ancien DRC était propre :

```text
- PMID caps collés au pin PMID/GND ;
- boucle PMID -> HSFET/SW -> L -> SYS -> caps -> GND minimale ;
- SYS caps collés au pin SYS ;
- CVBUS et CBAT au plus près de leurs pins ;
- REGN au plus près ;
- plusieurs vias GND/thermiques sous et autour du BQ ;
- SW très petit et éloigné de TS, I2C, CC, ADC ;
- cuivres BAT/SYS/MAIN/MODEM dimensionnés pour plusieurs ampères ;
- footprint BQ couvrant correctement toute la longueur des pads de puissance.
```

---

# 4. USB-C extérieur — sink/device uniquement

## 4.1 Fonction finale

```text
Charge 5 V
ADB
EDL
MTP / USB device
```

Aucun host USB-C, aucun DRP, aucun contrôleur CC, aucun PD V1.

## 4.2 Connecteur

Conserver :

```text
TYPE-C-31-M-12 / C165948
USB-C femelle 16 pins / USB2
Extended / Economic
```

Aucune référence Basic suffisamment convaincante n'a été retenue sans modifier le footprint/mécanique.

## 4.3 Data / VBUS

```text
A6+B6 -> D+ -> protection ESD -> Q6A OTG D+
A7+B7 -> D- -> protection ESD -> Q6A OTG D-
VBUS  -> TVS -> BQ VBUS
VBUS  -> anode B5819W ; cathode -> Q6A OTG VBUS
```

La Schottky empêche un 5 V que le Q6A produirait accidentellement en mode host de revenir vers le connecteur/BQ.

```text
D_USB_VBUS : B5819W SL / C8598
             SOD-123, 1 A, 40 V
             Basic
```

Le courant de cette diode n'est pas le courant système : elle sert à présenter le VBUS au port OTG Q6A en mode périphérique.

## 4.4 CC1 / CC2

```text
CC1 -> 5.1 kOhm -> GND / C25905 Basic
CC2 -> 5.1 kOhm -> GND / C25905 Basic
```

Ces deux Rd rendent le port définitivement sink/UFP.

Mesure fusionnée :

```text
CC1 -- 470 kOhm / C23178 --+
                            +--> CC_SENSE --> GP26 / ADC0
CC2 -- 470 kOhm / C23178 --+
                            |
                         100 nF / C1525
                            |
                           GND
```

Une seule CC est active selon l'orientation. L'autre revient au GND via son Rd ; le réseau donne environ la moitié de la tension CC active. Le chargement additionnel du CC actif reste négligeable par rapport à 5.1 kOhm.

Le condensateur 100 nF fournit une source basse impédance pendant l'échantillonnage ADC malgré les 470 kOhm. Firmware : attendre au moins ~100 ms après attach et moyenner plusieurs conversions. Toute valeur ambiguë = source Default, donc limite conservatrice.

## 4.5 Protection

Conserver :

```text
U_ESD  : SRV05-4 / C558418
         D+, D-, CC1, CC2
         Extended / Economic

D_VBUS : SMF5.0A / C193402
         TVS 5 V, SOD-123FL
         Extended / Economic
```

Aucun remplacement Basic vérifié n'offre un compromis suffisamment clair pour justifier un changement.

Pas de common-mode choke en V1.

## 4.6 Routage USB

Le routage de l'ancien prototype ne doit pas être copié tel quel. Pour USB-C -> Q6A :

```text
- paire D+/D- ~90 Ohm différentiel selon stack-up réel JLC ;
- même couche ;
- longueur appariée raisonnablement ;
- plan GND continu dessous ;
- le moins de vias possible ;
- aucun stub ;
- ESD au plus près du connecteur.
```

Q6A -> EC25 : câblage direct court, D+/D- torsadés ensemble, GND associé.

---

# 5. Alimentation RP2040-Tiny / Storage OFF

## 5.1 Rail MCU

```text
SYS -> TPS610995DRVR -> MCU_VSYS -> VSYS du RP2040-Tiny
```

Conserver :

```text
U_MCU_PWR : TPS610995DRVR / C2071098 / Extended
L_MCU     : MAKK2016T2R2M / C92923 / 2.2 uH / Extended
CIN       : 1 x 10 uF / C15850 Basic
COUT      : 2 x 10 uF / C15850 Basic
```

Le nom `3V6_MCU` est abandonné au profit de `MCU_VSYS` : TPS610995 est une version 3.6 V mais peut entrer en pass-through lorsque VIN dépasse suffisamment la cible. Ce comportement reste sûr car l'entrée VSYS du module RP2040-Tiny passe par son régulateur local et reste dans sa plage avec un SYS 1S.

Le composant est conservé pour deux raisons :

```text
- maintien d'une alimentation MCU exploitable lorsque la cellule descend ;
- true disconnection en shutdown pour le Storage OFF.
```

## 5.2 Storage switch

```text
SYS -> pad STORAGE_SW_1
pad STORAGE_SW_2 -> EN TPS610995
EN -> 100 kOhm -> GND
```

```text
switch fermé : MCU alimenté
switch ouvert: TPS610995 shutdown / MCU hard-off
```

Règle d'usage : Storage OFF uniquement après shutdown propre Q6A + modem. Si le switch est ouvert en fonctionnement, MAIN/MODEM sont encore maintenus environ 1 s puis retombent : ce n'est pas une procédure normale d'arrêt.

FPC du RP2040-Tiny laissé mécaniquement accessible pour USB/BOOTSEL/RUN ; aucun SWD supplémentaire n'est requis en V1.

---

# 6. MAIN_PWR et MODEM_PWR — OFF par défaut + maintien reset MCU

## 6.1 P-MOS de puissance

Conserver pour ces deux rails critiques :

```text
Q_MAIN_P / Q_MODEM_P : JMTQ55P02A / C2890429
                       P-MOS 20 V
                       ~12 mOhm @ |VGS|=2.5 V
                       Extended / Economic
```

AO3401A Basic n'est pas retenu ici : ~85 mOhm à 2.5 V créerait une chute et une dissipation trop fortes, particulièrement sur l'EC25 à ~2 A et à faible batterie.

## 6.2 Driver de grille

Pour MAIN et MODEM :

```text
P-MOS source -> SYS
P-MOS drain  -> charge
P-MOS gate   -> 100 kOhm -> SYS
P-MOS gate   -> drain AO3400A
AO3400A source -> GND

GPIO MCU -> 10 kOhm -> gate AO3400A
AO gate  -> 470 kOhm -> GND
AO gate  -> 2.2 uF -> GND
```

Composants :

```text
AO3400A : C20917 Basic
10 kOhm : C25744 Basic
470 kOhm: C23178 Basic
2.2 uF  : C23630 Basic, 16 V X5R 0603
100 kOhm P-gate->SYS : C25741 Basic
```

Principe :

```text
GPIO HIGH : AO3400A ON -> gate P-MOS basse -> rail ON
GPIO LOW  : rail OFF rapidement
GPIO Hi-Z/reset : C_HOLD se décharge via 470 kOhm -> maintien temporaire
```

Constante RC nominale :

```text
470 kOhm x 2.2 uF ~= 1.03 s
```

Le délai électrique utile dépend du seuil réel AO3400A ; cible pratique ~0.8 à 1.5 s. Il doit être mesuré sur prototype.

Un arrêt volontaire reste rapide car le GPIO force LOW à travers 10 kOhm ; le condensateur ne ralentit donc pas l'extinction d'environ une seconde.

## 6.3 Comportement firmware au reset

Très important : au tout début du boot RP2040, GP10 et GP29 sont d'abord lus comme entrées.

```text
si GP10 est encore HIGH par le RC -> le firmware le passe immédiatement en sortie HIGH
si GP29 est encore HIGH par le RC -> idem
```

Ainsi, un reset MCU court ne coupe ni Android ni le modem. Le HEARTBEAT Q6A sert ensuite à vérifier la cohérence système, mais n'est pas nécessaire pour survivre aux premières millisecondes du reboot MCU.

---

# 7. ANNEXE1 / ANNEXE2

Pour économiser deux composants Extended :

```text
Q_A1_P / Q_A2_P : AO3401A / C15127 Basic
                   P-MOS 30 V, ~85 mOhm @ 2.5 V
```

Limite de conception V1 :

```text
1 A continu cible par sortie
```

À 1 A, la dissipation résistive reste de l'ordre de ~85 mW typique au MOSFET. Ce choix convient à l'ampli audio et aux accessoires modérés.

Commande :

```text
source P-MOS -> SYS
drain -> ANNEXE+
gate -> 100 kOhm -> SYS
gate -> drain AO3400A C20917
AO source -> GND
GPIO -> 10 kOhm -> gate AO
AO gate -> 100 kOhm -> GND
```

Pas de condensateur de maintien sur ANNEXE1/2 : elles sont OFF immédiatement/default pendant reset MCU.

---

# 8. Q6A <-> MCU

## 8.1 Signaux retenus

```text
Q6A J20 pin 36 / GPIO59 -> Q6A_HEARTBEAT -> GP27
Q6A J20 pin 37 / GPIO58 <- Q6A_SLEEP_REQ <- GP9
Q6A J20 pin 3  / GPIO24 / I2C6_SDA <-> GP14 SBS_SDA
Q6A J20 pin 5  / GPIO25 / I2C6_SCL <-> GP15 SBS_SCL
Q6A J20 pin 1 ou 17 / 3V3 -> Q6A_3V3_REF
Q6A GND -> GND
Q6A PWR_ON_KEY <- S8050 <- GP8
```

## 8.2 SLEEP_REQ

Le signal est actif bas et utilisé comme open-drain logiciel :

```text
Q6A_3V3 -> 100 kOhm -> SLEEP_REQ_Q6A
SLEEP_REQ_Q6A -> 10 kOhm série -> GP9
```

Firmware RP2040 :

```text
actif   : GP9 sortie LOW
relâché : GP9 entrée / Hi-Z
```

Le pull-up élevé limite le back-power lorsque le RP est hors tension.

## 8.3 HEARTBEAT

Le heartbeat est lent ; priorité à la limitation de back-power :

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Ainsi :

```text
Q6A OFF       -> LOW défini
Q6A heartbeat -> HIGH/LOW reçu sans charge significative
RP OFF        -> courant parasite limité à quelques dizaines de uA maximum
```

Le signal ne commande jamais directement une coupure brutale. Perte heartbeat hors procédure connue -> tentative de réveil/récupération, puis power-cycle seulement après timeout.

## 8.4 PWR_ON_KEY Q6A

```text
GP8 -> 10 kOhm -> base S8050 C2146 Basic
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Fonctions :

```text
cold boot après activation MAIN_PWR
wake depuis deep
événement bouton Power Android
appui long de récupération si nécessaire
```

Le bouton Power utilisateur ne va donc pas directement au Q6A ; il est lu par le MCU.

## 8.5 SBS / batterie virtuelle

```text
GP14 <-> Q6A I2C6_SDA
GP15 <-> Q6A I2C6_SCL
```

Prévoir sur notre PCB :

```text
2 x 4.7 kOhm / C23162 Basic vers Q6A_3V3
DNP par défaut
```

On ne les peuple que si l'inspection/mesure confirme que le Q6A n'a pas déjà ses pull-up utiles sur ce bus. Les pull-up ne doivent pas aller vers 3V3_MCU, afin d'éviter d'alimenter le Q6A lorsque MAIN_PWR est OFF.

Le SBS est une interface logicielle V1, pas une exigence de boot matériel. Une erreur de driver Android ne doit pas empêcher la carte de démarrer.

---

# 9. EC25 — alimentation et USB

## 9.1 Alimentation principale

```text
SYS -> MODEM_PWR -> EC25 carrier BAT
GND -> EC25 carrier GND
```

Le pin `BAT` du carrier est la puissance principale du modem. `USB_VBUS` n'est pas la puissance LTE principale.

MODEM_PWR est coupé par défaut et doit respecter la fenêtre VSYS définie plus haut.

## 9.2 USB direct Q6A

```text
Q6A USB2 host VBUS -> EC25 VBUS
Q6A USB2 host D+   -> EC25 DP
Q6A USB2 host D-   -> EC25 DN
Q6A GND            -> EC25 GND
```

Aucun de ces quatre fils ne transite par la carte Power V1.

Pas de switch VBUS EC25 en V1. Cette décision impose une gate : le Q6A doit réellement mettre ce port USB en suspend pendant le deep sleep. Quectel indique que, si le host ne sait pas suspendre l'USB, il faut couper USB_VBUS pour atteindre le sommeil modem. Si cette gate échoue, le câble direct devra recevoir ultérieurement une coupure VBUS ou la carte devra évoluer.

## 9.3 Séquence modem

Démarrage :

```text
1. vérifier 3.35 <= VSYS <= 4.20 V
2. MODEM_PWR = ON
3. attendre >= 30 ms
4. tirer PWRKEY bas >= 500 ms
5. attendre VIO/UART/USB
```

Extinction normale :

```text
AT+QPOWD en priorité
ou PWRKEY bas >= ~650 ms
attendre confirmation / disparition VIO/USB
puis MODEM_PWR OFF
```

Coupure MODEM_PWR forcée uniquement pour recovery.

## 9.4 Sleep modem

```text
AT+QSCLK=1
DTR HIGH
USB host Q6A suspendu
```

DTR LOW permet le réveil par UART ; trafic USB peut également réveiller le module selon la pile host.

---

# 10. EC25 <-> MCU logique 1.8 V

## 10.1 Level-shifter

Conserver :

```text
SN74AVC4T245DR / C22495 / Extended
VCCA = EC25 VIO (~1.8 V)
VCCB = 3V3_MCU
```

Le composant possède deux groupes de deux bits avec DIR/OE indépendants :

```text
groupe 1, DIR A->B : EC25 TXD + RI -> MCU

groupe 2, DIR B->A : MCU TX vers EC25 RXD + MCU DTR -> EC25
```

`1OE` et `2OE` sont maintenus LOW par câblage/pulldown afin que le traducteur soit actif lorsque ses deux alimentations existent. La fonction VCC isolation/Ioff protège les ports lorsque VCCA ou VCCB tombe à 0 V.

Ajouter :

```text
VIO -> 100 kOhm -> GND
EC25 DTR -> 100 kOhm -> VIO
EC25 RXD -> 100 kOhm -> VIO
```

Le pull-up DTR donne un état sleep-friendly quand le MCU/translator est indisponible. Le pull-up RXD maintient l'entrée UART EC25 au niveau idle au lieu de la laisser flottante.

Découplage :

```text
100 nF VCCA-GND au plus près
100 nF VCCB-GND au plus près
```

## 10.2 EC25 PWRKEY

```text
GP13 -> 10 kOhm -> base S8050 C2146 Basic
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> EC25 PWRKEY
```

Aucun pull-up 3.3 V ajouté côté collecteur.

## 10.3 EC25 RESET_N / GP28

RESET_N est supprimé des fonctions normales. GP28 devient réserve V1.

```text
GP28 -> TP_GP28_3V3
```

Prévoir un chemin DNP :

```text
GP28 -> 10 kOhm DNP -> base S8050 DNP
base -> 100 kOhm DNP -> GND
émetteur -> GND
collecteur -> TP_EC25_RST_OD -> EC25 RESET_N via strap/pad DNP
```

Cela fournit :

```text
- un GPIO 3.3 V brut sur TP_GP28_3V3 ;
- une sortie open-drain compatible domaine 1.8 V sur TP_EC25_RST_OD si composants peuplés.
```

Ce n'est pas un GPIO push-pull 1.8 V. RESET_N reste un outil de dépannage, derrière AT/PWRKEY/power-cycle dans l'ordre de récupération.

---

# 11. Pinout RP2040-Tiny final

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
GP26  <-  USB_CC_SENSE / ADC0
GP27  <-  Q6A HEARTBEAT
GP28  <-> RESERVE / TP 3.3 V / option EC25 RESET_N open-drain DNP
GP29  ->  MODEM_PWR_EN avec RC hold
```

Toutes les GPIO exposées par le RP2040-Tiny sont affectées, mais GP28 reste volontairement disponible comme réserve de dépannage.

Bouton Power utilisateur :

```text
GP7 -> 10 kOhm pull-up vers 3V3_MCU
bouton NO entre GP7 et GND
```

---

# 12. Machine d'états puissance figée

## 12.1 Cold boot

```text
STORAGE_SW fermé
-> MCU démarre
-> initialise BQ : VREG 4.15, VSYSMIN 3.84, limites source
-> si demande de démarrage utilisateur : GP10 HIGH
-> MAIN_PWR ON
-> attendre ~100 à 300 ms
-> pulse Q6A PWR_ON_KEY ~200 à 300 ms, à ajuster au banc
-> attendre HEARTBEAT
-> si VSYS dans fenêtre modem : GP29 HIGH
-> attendre >=30 ms
-> EC25 PWRKEY >=500 ms
```

## 12.2 Reset MCU en fonctionnement

```text
RC MAIN/MODEM maintient les AO3400A actifs ~1 s
-> firmware lit GP10/GP29 immédiatement
-> s'ils sont encore hauts, les force HIGH
-> Q6A et modem ne subissent aucune coupure
-> HEARTBEAT et états BQ/EC25 servent ensuite à resynchroniser la machine d'états
```

## 12.3 Suspend normal

```text
appui utilisateur -> MCU
-> SLEEP_REQ actif
-> service Q6A : prépare EC25, DTR/QSCLK, suspend USB modem, met OTG externe sur none,
   traite Wi-Fi et autres wake sources, indique transition via heartbeat
-> Q6A entre mem_sleep=deep
-> MAIN_PWR reste ON
-> MODEM_PWR reste ON si modem doit rester joignable/sleep
-> HEARTBEAT peut s'arrêter volontairement
```

Réveil : pulse Q6A PWR_ON_KEY.

## 12.4 Shutdown normal

```text
1. demande shutdown Android propre ;
2. arrêt propre EC25 ;
3. attendre confirmations/timeouts ;
4. MODEM_PWR OFF ;
5. MAIN_PWR OFF ;
6. MCU reste alimenté tant que STORAGE_SW est fermé.
```

## 12.5 Storage OFF

Uniquement après shutdown propre : ouvrir STORAGE_SW. Le TPS610995 isole le MCU. Tous les rails contrôlés sont déjà OFF ; aucun maintien permanent n'est nécessaire.

---

# 13. SOC / batterie virtuelle

Le MAX17048 est supprimé de la V1 finale afin de réduire BOM, surface et complexité.

Le MCU utilise :

```text
VBAT_ADC
VSYS_ADC
IBAT_ADC
IBUS_ADC
TS_ADC
état de charge / faults BQ
courbe OCV/SOC de la cellule
intégration logicielle du courant quand la mesure BQ est valide
```

Limite assumée : le BQ25628E n'est pas un fuel gauge à modèle de cellule. Le SOC exposé à Android via SBS sera une estimation V1 et devra être calibré. Ce point n'affecte pas la sécurité de charge.

---

# 14. Connectique finale

Puissance :

```text
J_BAT : Molex Micro-Fit 436500300 / C503478
        BAT+ / GND / NTC

J_Q6A : Molex Micro-Fit 436500200 / C192562
        MAIN_PWR+ / GND

J_A1  : Molex Micro-Fit 436500200 / C192562
        ANNEXE1+ / GND

J_A2  : Molex Micro-Fit 436500200 / C192562
        ANNEXE2+ / GND
```

MODEM_PWR : deux gros pads THT BAT/GND sur la carte, câblage direct vers le carrier EC25. Pas de Micro-Fit supplémentaire par défaut afin d'économiser l'encombrement.

Logique Q6A / EC25 : pads/trous THT clairement sérigraphiés pour harnais direct.  
STORAGE_SW : deux pads THT accessibles.  
RP2040-Tiny : face BOTTOM, soudé manuellement après PCBA.

---

# 15. Testpoints obligatoires

```text
VBUS_USB_C
BQ_VBUS
BAT
SYS
MCU_VSYS
3V3_MCU
TPS610995_EN
GND

BQ_SDA
BQ_SCL
BQ_INT
BQ_TS
BQ_ILIM

MAIN_PWR_GATE
MAIN_PWR_OUT
MODEM_PWR_GATE
MODEM_PWR_OUT
ANNEXE1_OUT
ANNEXE2_OUT

CC1
CC2
CC_SENSE

Q6A_3V3_REF
Q6A_PWRKEY
Q6A_SLEEP_REQ
Q6A_HEARTBEAT
SBS_SDA
SBS_SCL

EC25_VIO
EC25_TXD
EC25_RXD
EC25_DTR
EC25_RI
EC25_PWRKEY
TP_GP28_3V3
TP_EC25_RST_OD
```

---

# 16. PCB / stack-up / placement

```text
PCB 4 couches
max strict : 70 x 25 mm
RP2040-Tiny : BOTTOM
```

Organisation recommandée :

```text
L1 : composants + USB2 + boucles puissance locales
L2 : GND continu
L3 : puissance / signaux lents ; SW BQ uniquement selon layout TI
L4 : signaux + GND
```

Contraintes :

```text
- BQ placé/routé avant le reste ;
- aucun signal sensible sous/près de SW ;
- plans GND continus pour USB ;
- vias thermiques BQ ;
- BAT/SYS/MAIN/MODEM très larges et via-stitchés si changement de couche ;
- connecteurs mécaniquement accessibles ;
- footprint RP2040 vérifié à l'échelle 1:1 sur module réel ;
- aucun via/testpoint susceptible de toucher le dessous du module ;
- FPC RP2040 accessible après assemblage ;
- testpoints accessibles sans démonter le module.
```

---

# 17. BOM — décisions JLCPCB après revue Basic

## 17.1 Extended conservés

```text
BQ25628ERYKR       C18221178   chargeur/power-path
XRIM252012S1R0MBCA C22471110   inductance BQ 1 uH
TPS610995DRVR      C2071098    alimentation MCU / true shutdown
MAKK2016T2R2M      C92923      inductance TPS 2.2 uH
SN74AVC4T245DR     C22495      translation 1.8/3.3 V EC25
JMTQ55P02A         C2890429    P-MOS MAIN_PWR et MODEM_PWR, x2
TYPE-C-31-M-12     C165948     USB-C
SRV05-4            C558418     ESD D+/D-/CC1/CC2
SMF5.0A            C193402     TVS VBUS
```

Pas de remplacement Basic retenu pour ces fonctions critiques.

## 17.2 Basic utilisés / ajoutés

```text
AO3400A        C20917   NMOS drivers de P-MOS
AO3401A        C15127   P-MOS ANNEXE1/2
S8050          C2146    Q6A PWRKEY + EC25 PWRKEY + option RESET DNP
B5819W SL      C8598    isolation VBUS vers Q6A

5.6 kOhm       C23189   ILIM BQ
5.1 kOhm       C25905   CC Rd + réseau NTC
30 kOhm        C22984   NTC
4.7 kOhm       C23162   pull-up SBS DNP
10 kOhm        C25744   divers
100 kOhm       C25741   divers
470 kOhm       C23178   RC hold + CC merge
1 MOhm         C22935   pulldown heartbeat
1 kOhm         C11702   uniquement là où une petite série reste utile ; ne pas l'utiliser sur heartbeat

100 nF        C1525
1 uF          C52923
2.2 uF 16 V   C23630
4.7 uF        C1779
10 uF         C15850
47 nF 50 V    C1622
```

Les quantités exactes doivent être générées depuis le schéma V0.5 après application de ces règles ; ne pas recopier les quantités de la BOM du 30/09, devenues obsolètes.

---

# 18. Fonctions explicitement supprimées / non retenues

```text
USB-C host / DRP / contrôleur CC   -> supprimé V1
USB-PD / 9 V                       -> supprimé V1
Q6A GPIO59 WAKE                    -> remplacé par vrai PWR_ON_KEY ; GPIO59 = HEARTBEAT
CC1 et CC2 sur deux ADC            -> fusionnés sur GP26
EC25 alimenté par VBUS Q6A         -> supprimé ; BAT depuis MODEM_PWR
USB EC25 à travers PCB Power       -> supprimé ; câblage direct
switch USB_VBUS EC25               -> non monté V1, validation suspend host obligatoire
EC25 RESET dédié                   -> supprimé ; option DNP sur GP28 seulement
MAX17048                           -> supprimé V1
DISPLAY_PWR                        -> absent de cette carte
codec audio Minimal EC25           -> supprimé
jack utilisateur                   -> supprimé
mux audio analogique               -> supprimé
CMC USB2                           -> non monté V1
fusible BAT assemblé               -> DNP + jumper cuivre par défaut
```

---

# 19. Gates avant fabrication — BLOQUANTES

Le CDC électrique est figé, mais la commande de fabrication reste conditionnée à ces validations :

```text
[ ] Q6A J19 : R7 retiré, R24 2 mOhm final, R190 100 k, R191 10 k, FB4 DNP,
    R185..R189 DNP confirmés.

[ ] Q6A alimenté par J19 au banc : 4.2 / 3.8 / 3.4 / 3.0 V ; boot, charge CPU, idle, deep.

[ ] PWR_ON_KEY réel : cold boot et wake deep validés après modification J19.

[ ] GPIO58 SLEEP_REQ déclenche la procédure de deep voulue ; GPIO59 fonctionne comme sortie heartbeat.

[ ] EC25 : VIO/TXD/RI mesurés sur carrier réel avant connexion définitive.

[ ] EC25 alimenté depuis SYS/MODEM_PWR : mesurer BAT carrier à SYS haut ; jamais >=4.30 V.

[ ] EC25 : vérifier démarrage et LTE sous pics avec MODEM_PWR ; observer VSYS à l'oscilloscope.

[ ] EC25 USB : vérifier que le contrôleur host Q6A suspend réellement le bus et que le modem atteint son sommeil
    avec VBUS restant connecté. Si échec, il faut ajouter une coupure VBUS dans le câblage ou réviser le PCB.

[ ] Pack batterie choisi : capacité de courant PCM/cellule documentée, câbles/connecteur adaptés.

[ ] TPS610995 : true shutdown / courant Storage OFF / absence de back-power validés sur schéma puis prototype.

[ ] SBS I2C : vérifier présence/absence des pull-up Q6A avant de peupler les 4.7 k DNP.

[ ] Footprint RP2040-Tiny mesuré + impression 1:1 + FPC accessible.

[ ] Layout BQ comparé visuellement à la recommandation TI ; boucle SW/PMID/SYS revue avant DRC final.

[ ] Routage USB-C D+/D- recalculé avec le stack-up JLC choisi ; vraie paire différentielle.

[ ] Analyse back-power carte OFF : Q6A, EC25, BQ, SBS, heartbeat, USB et RP2040.

[ ] ERC = 0 erreur critique ; DRC = 0 erreur ; polarités/footprints/PnP/BOM contrôlés manuellement.

[ ] PCB <= 70 x 25 mm et aucune collision mécanique BOTTOM/TOP.
```

---

# 20. Tests après fabrication

```text
[ ] USB-C 5 V -> BQ -> SYS sans batterie
[ ] batterie seule -> SYS
[ ] plug/unplug USB sans reboot
[ ] ILIM matériel ~0.45 A
[ ] CC_SENSE : Default / 1.5 A / 3 A correctement classés
[ ] IINDPM / EN_EXTILIM séquencés correctement
[ ] charge 0.5 / 1 / 1.5 / 2 A thermiquement caractérisée
[ ] NTC bloque/limite correctement hors plage
[ ] télémétrie BQ cohérente
[ ] VREG 4.15 V et VSYSMIN 3.84 V programmés au boot
[ ] RP stable sur toute la plage SYS ; MCU_VSYS mesuré en boost/down/pass-through
[ ] Storage OFF : courant résiduel mesuré
[ ] MAIN_PWR et MODEM_PWR OFF par défaut
[ ] reset MCU : MAIN/MODEM ne chutent pas pendant reboot court
[ ] arrêt volontaire : MAIN/MODEM chutent rapidement malgré C_HOLD
[ ] ANNEXE1/2 OFF par défaut, <=1 A continu validé
[ ] Q6A PWRKEY cold boot / wake / appui long
[ ] SLEEP_REQ + heartbeat
[ ] SBS/I2C sans back-power
[ ] USB-C ADB/EDL/MTP
[ ] aucun retour VBUS Q6A vers USB-C grâce à la Schottky
[ ] EC25 BAT dans 3.35..4.20 V en fonctionnement nominal
[ ] EC25 USB direct Q6A stable
[ ] EC25 UART/DTR/RI via SN74AVC4T245
[ ] EC25 sleep avec USB suspend
[ ] EC25 PWRKEY / shutdown / recovery
[ ] consommation deep mesurée
[ ] consommation storage-off mesurée
```

---

# 21. Conclusion de gel

Après audit bloc par bloc, aucune incohérence électrique structurelle ne justifie d'ajouter un nouveau IC ou de changer l'architecture.

Le schéma V0.5 doit être construit exactement à partir de ce document. Les points encore ouverts sont des **mesures/gates de validation physique**, pas des choix d'architecture, à l'exception explicite du cas où le Q6A ne réussirait pas à suspendre l'USB de l'EC25 avec VBUS connecté.

Après validation utilisateur de ce CDC, il pourra remplacer `CDC_CARTE_POWER_MCU_V1_2026-09-30.md` comme référence courante du dépôt.

---

# 22. Sources primaires utilisées pour l'audit

- Texas Instruments — `BQ25628E I2C Controlled 1-Cell, 2A, Maximum 18V Input, Buck Battery Charger with NVDC Power Path Management`, Rev. C, 2025.
- Texas Instruments — `TPS61099x` datasheet / product documentation.
- Texas Instruments — `SN74AVC4T245`, Rev. I, 2025.
- Quectel — `EC25 Series Hardware Design`, V2.4.
- Radxa — Dragon Q6A V1.21 schematic + GPIO/USB documentation.
- Waveshare — RP2040-Tiny schematic / mechanical documentation.
- JLCPCB/LCSC part library — classifications Basic/Extended rechecked 2026-10-02 for the newly selected parts.
