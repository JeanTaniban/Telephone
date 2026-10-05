# CDC — Carte Power / MCU V1 du Maker Phone Q6A — CURRENT

**Date :** 2026-10-05  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Révision :** **V0.8 — architecture courante**  
**Fabrication cible :** JLCPCB Economic PCBA  
**Priorité composants :** Basic > Promotional > Extended ; stock et classification à revalider au moment de la commande.

Ce document est la **référence électrique normative courante** de la carte Power/MCU. Il supersède les anciens CDC Power/MCU dès qu'ils sont contradictoires.

Le point **tension de charge JK50 / alimentation principale EC25** reste en étude : tant qu'un autre rail modem n'est pas validé, la baseline électrique reste `SYS -> MODEM_PWR -> EC25 BAT` avec une politique de charge conservatrice autour de `VREG = 4.15 V`. Ne jamais passer la JK50 à 4.40 V avec l'EC25 alimenté directement par SYS sans nouvelle validation d'architecture.

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
 |       +-- EN <- STORAGE_SW ; EN pulldown 100 kOhm                         |
 |                                                                            |
 +--> MAIN_PWR : JMTQ55P02A --> Q6A J19                                      |
 |       commande AO3400A + maintien RC ~1 s                                 |
 |                                                                            |
 +--> MODEM_PWR : JMTQ55P02A --> EC25 carrier BAT                            |
 |       commande AO3400A + maintien RC ~1 s                                 |
 |                                                                            |
 +--> ANNEXE1 : AO3401A + driver AO3400A -> ampli HP                         |
 +--> ANNEXE2 : AO3401A + driver AO3400A -> annexes                          |

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
   <- Q6A HEARTBEAT/STATE via GPIO59
  <-> EC25 UART/DTR/RI via SN74AVC4T245
   -> EC25 PWRKEY via S8050
   -> MAIN_PWR / MODEM_PWR / EC25_USB_VBUS / ANNEXE1 / ANNEXE2
   <- bouton Power utilisateur
   <- CC_SENSE analogique fusionné

Boutons Volume
  VOL+ -> Q6A GPIO31 direct
  VOL- -> Q6A GPIO30 direct
  aucune liaison vers RP2040
```

La carte Power ne transporte ni le DSI écran ni le tactile. Pour l'USB Q6A <-> EC25, **seul le 5 V VBUS traverse désormais la carte Power afin d'être commutable** ; D+/D- restent en faisceau court direct Q6A <-> carrier EC25.

---

# 1. Batterie JK50, connecteur, protection et BAT_ARM

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

La vraie batterie reçue doit être mesurée avant gel mécanique.

Baseline actuelle tant que MODEM_PWR reste directement issu de SYS :

```text
VREG initial : ~4.15 V
ICHG         : <=2 A, limité thermiquement et selon charge système
```

La tension de charge finale n'est pas considérée définitivement gelée tant que l'alimentation EC25 n'a pas été réétudiée.

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

Le footprint et l'orientation proviennent exclusivement du drawing Molex officiel. ID1/ID2 restent en haute impédance/testpoints en V1.

## 1.3 NTC

```text
NTC1 -> TP -> strap 0 Ohm/DNP -> BQ_TS
NTC2 -> TP -> strap 0 Ohm/DNP -> BQ_TS
```

Une seule source thermique est sélectionnée. La courbe R/T réelle de la JK50 doit être caractérisée avant de figer RT1/RT2. Prévoir pads `NTC_EXT/GND` de secours si placement gratuit.

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

Le seuil exact OCP du PCM JK50 doit être caractérisé au banc. `4.85 A` est une valeur de décharge maximale documentée, pas un seuil OCP supposé.

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

## 2.2 Initialisation firmware obligatoire

Au **cold boot système**, MAIN_PWR, MODEM_PWR, EC25_USB_VBUS, ANNEXE1 et ANNEXE2 restent OFF jusqu'à validation du BQ.

Séquence normative :

```text
1. Initialiser immédiatement les GPIO dans leurs états sûrs.
2. Vérifier la présence I2C du BQ.
3. Dès le passage en host mode, désactiver explicitement le watchdog BQ
   sauf si une politique de refresh volontaire est ultérieurement validée.
4. Programmer les paramètres critiques : VREG, VSYSMIN, ICHG, IINDPM/ILIM,
   EN_EXTILIM, ADC et politique TS/thermique.
5. Relire les registres critiques.
6. Si readback incohérent ou BQ absent : rester en OFF/FAULT ; ne pas booter le Q6A.
7. Si configuration valide : MAIN_PWR peut être autorisé.
```

Valeurs de départ :

```text
RILIM               : 5.6 kOhm / C23189 Basic / 1 %
VREG baseline       : ~4.15 V tant que EC25 est alimenté directement depuis SYS
VSYSMIN             : 3.84 V cible
source inconnue     : IINDPM conservateur ~0.44 A
Rp 1.5 A reconnu    : ~1.35 A max de départ
ICHG                : <=2 A
```

Le réglage `IBAT_PK`, `TREG` et `ITERM` n'est pas gelé : il sera décidé après caractérisation thermique et mesure des pointes système/JK50.

## 2.3 Reset MCU pendant RUN

Le cas reset MCU pendant que Q6A/modem sont déjà alimentés est distinct du cold boot :

```text
1. Les RC de hold maintiennent temporairement MAIN_PWR, MODEM_PWR et EC25_USB_VBUS.
2. Au tout début du firmware : réaffirmer GP10, GP29 et GP6 selon l'état récupéré attendu.
3. Ensuite seulement : relecture/configuration BQ et resynchronisation HEARTBEAT/UART.
```

Aucun reset MCU court ne doit provoquer involontairement un power-cycle Q6A/modem.

## 2.4 TS / thermique

Topologie configurable pour JK50. Anciennes valeurs de référence 103AT-2 uniquement à titre de fallback :

```text
RT1 = 5.1 kOhm / C25905 Basic
RT2 = 30 kOhm  / C22984 Basic
```

Ne pas les considérer comme validées pour JK50 avant mesure.

## 2.5 Passifs principaux

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

Layout strict selon TI : boucles PMID/SW/SYS minimales, caps collés aux pins, SW compact, plan GND continu, vias thermiques, TS/I2C/ADC éloignés du nœud SW.

---

# 3. USB-C extérieur — charge + device uniquement

```text
Fonctions : charge 5 V, ADB, EDL, MTP/USB device
Pas de host USB-C, pas de DRP, pas de PD V1.
```

Connecteur :

```text
TYPE-C-31-M-12 / C165948
A6+B6 -> D+ -> SRV05-4 -> Q6A OTG D+
A7+B7 -> D- -> SRV05-4 -> Q6A OTG D-
VBUS  -> SMF5.0A -> BQ VBUS
VBUS  -> B5819W SL -> Q6A OTG VBUS
CC1   -> 5.1 kOhm -> GND
CC2   -> 5.1 kOhm -> GND
CC1/CC2 -> 470 kOhm chacun -> CC_SENSE -> GP26/ADC0
CC_SENSE -> 100 nF -> GND
```

D+/D- : paire USB2 ~90 Ohm différentiel sur stack-up réel, plan GND continu, aucun stub inutile, minimum de vias.

USB-C branché + MAIN_PWR OFF est un état à valider au banc ; une alimentation partielle interne du Q6A par USB n'est problématique que si elle crée une consommation OFF excessive, un rail indésirable ou un comportement de reboot incorrect.

---

# 4. RP2040-Tiny / always-on / Storage OFF

```text
Module : Waveshare RP2040-Tiny soudé manuellement BOTTOM, sans headers
SYS -> TPS610995DRVR / C2071098 -> MCU_VSYS -> VSYS RP2040-Tiny
L_MCU : MAKK2016T2R2M / C92923 / 2.2 uH
CIN   : 10 uF
COUT  : 2 x 10 uF
```

Storage :

```text
SYS -> STORAGE_SW -> EN TPS610995
EN -> 100 kOhm -> GND
fermé  : MCU alimenté
ouvert : hard-off MCU / stockage
```

Le FPC du RP2040-Tiny reste accessible pour USB/BOOTSEL/RUN. Footprint réel contrôlé + impression 1:1 + keepout sous module.

## 4.1 Pinout RP2040-Tiny

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
GP28  <-> RESERVE / TP 3.3 V / option EC25 RESET_N open-drain DNP
GP29  ->  MODEM_PWR_EN avec RC hold
```

GP6 n'est plus une réserve : il pilote indépendamment le 5 V USB du modem. GP28 reste la réserve principale.

---

# 5. MAIN_PWR et MODEM_PWR

## 5.1 Puissance

```text
Q_MAIN_P  : JMTQ55P02A / C2890429
Q_MODEM_P : JMTQ55P02A / C2890429
```

Baseline : **un seul P-MOS par rail**. Aucun montage back-to-back n'est imposé tant qu'un problème réel de retour de courant n'est pas démontré au banc.

## 5.2 Driver commun et hold

Par rail :

```text
P-MOS source -> SYS
P-MOS drain  -> charge
P-MOS gate   -> 100 kOhm -> SYS
P-MOS gate   -> drain AO3400A
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

Au reset MCU, GP10/GP29 sont prioritaires avant toute tâche non critique.

---

# 6. ANNEXE1 / ANNEXE2 — commande corrigée

Les annexes utilisent le même **principe de driver low-side** que MAIN/MODEM pour garantir un vrai OFF, mais sans condensateur de hold.

Pour chaque annexe :

```text
SYS -> source AO3401A / C15127 Basic
AO3401A drain -> ANNEXE_OUT
AO3401A gate -> 100 kOhm -> SYS
AO3401A gate -> drain AO3400A / C20917 Basic
AO3400A source -> GND
GPIO GP11 ou GP12 -> 10 kOhm -> gate AO3400A
gate AO3400A -> 100 kOhm -> GND
PAS de 1 uF de hold
```

Ainsi : GPIO absent/Hi-Z/reset -> annexe OFF de manière déterministe. Aucune nouvelle référence JLC n'est ajoutée.

```text
ANNEXE1 : alimentation commutée ampli haut-parleur, cible locale ~1 A
ANNEXE2 : SYS commuté générique caméras / flash / capteurs / outils, cible locale ~1 A
```

Toute charge nécessitant 5 V, 3.3 V fixe ou une autre tension doit avoir sa propre régulation aval.

---

# 7. Q6A <-> carte Power / MCU

## 7.1 Liaisons

```text
Q6A J20 pin 36 / GPIO59 -> HEARTBEAT_STATE -> GP27
Q6A J20 pin 37 / GPIO58 <- SHUTDOWN_REQ <- GP9
Q6A J20 pin 3 / GPIO24 / I2C6_SDA <-> GP14 SBS_SDA
Q6A J20 pin 5 / GPIO25 / I2C6_SCL <-> GP15 SBS_SCL
Q6A J20 pin 1 ou 17 / 3V3 -> Q6A_3V3_REF
Q6A GND -> GND
Q6A PWR_ON_KEY <- S8050 <- GP8
MAIN_PWR_OUT -> Q6A J19+
GND -> Q6A J19-
```

Toutes ces liaisons sont sur pastilles THT pour fils soudés.

## 7.2 J19 Q6A

```text
R7       -> DNP/retiré
R24      -> 2 mOhm 1 %
R190     -> 100 kOhm GND
R191     -> 10 kOhm GND
FB4      -> DNP
R185..189 -> DNP si inspection confirme
```

Validation J19 : alimentation externe 4.2 / 3.8 / 3.4 / 3.0 V, boot/idle/charge/deep.

## 7.3 PWR_ON_KEY

```text
GP8 -> 10 kOhm -> base S8050
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Fonctions : cold boot, événement Power Android, wake depuis suspend, récupération par maintien long si nécessaire. Pulse court initial à caractériser autour de 100–300 ms.

## 7.4 SHUTDOWN_REQ — remplace SLEEP_REQ

```text
Q6A_3V3 -> 100 kOhm -> SHUTDOWN_REQ
SHUTDOWN_REQ -> 10 kOhm série -> GP9
MCU actif : sortie LOW
repos      : input/Hi-Z
```

**SHUTDOWN_REQ ne commande jamais un suspend-to-RAM.** Il signifie uniquement : demander à Android/Linux une **extinction complète propre**.

Un service/daemon côté Q6A doit convertir cette requête en chemin de shutdown Android normal.

## 7.5 HEARTBEAT_STATE

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

GPIO59 est un canal de statut logiciel. Il doit pouvoir distinguer au minimum :

```text
RUN                  : Q6A vivant / fonctionnement normal
SHUTDOWN_IN_PROGRESS : arrêt propre en cours
SHUTDOWN_READY       : écritures terminées, MAIN peut être coupé
AUTO_OFF_ALLOWED     : Android autorise le MCU à démarrer/continuer un timer d'extinction automatique
```

Le codage exact par niveaux/pulses reste firmware, mais la sémantique est normative.

Une disparition du heartbeat seule ne provoque jamais une coupure immédiate : un suspend Android peut arrêter le logiciel de heartbeat.

## 7.6 SBS

```text
GP14 <-> Q6A I2C6_SDA
GP15 <-> Q6A I2C6_SCL
2 x 4.7 kOhm vers Q6A_3V3 : DNP par défaut
```

Ne jamais tirer SBS vers 3V3_MCU. MCU en Hi-Z côté SBS quand Q6A est OFF jusqu'à validation de l'absence de back-power.

---

# 8. Boutons utilisateur

## 8.1 Power

```text
GP7 <- bouton NO vers GND
pull-up 10 kOhm vers 3V3_MCU
```

Politique :

```text
appui court : MCU génère un pulse court PWR_ON_KEY ; Android gère écran, lock, wakelocks et autosuspend.
appui long  : MCU demande d'abord un shutdown propre via SHUTDOWN_REQ.
si aucun progrès après timeout : tentative de long PWR_ON_KEY à caractériser.
si système toujours bloqué après timeout de récupération : coupure forcée en dernier recours.
```

Le MCU n'essaie pas de décider si musique/appel/GPS/téléchargement autorisent un suspend. Android est seul maître du suspend normal.

## 8.2 Volume

```text
VOL+ : Q6A J20 pin 29 / GPIO31
VOL- : Q6A J20 pin 32 / GPIO30
```

Pour chaque bouton : Q6A_3V3 -> 10 kOhm -> GPIO -> bouton NO -> GND. Entrées actives bas, `gpio-keys` Android/Linux.

---

# 9. EC25 — alimentation, USB, supervision

## 9.1 Alimentation principale

Baseline actuelle :

```text
SYS -> MODEM_PWR -> EC25 carrier BAT
GND -> EC25 carrier GND
```

MODEM_PWR et GND utilisent de grosses pastilles THT. EC25 BAT doit rester dans sa plage sûre ; tant que ce chemin direct est utilisé, ne pas augmenter la charge JK50 vers 4.40 V sans revalidation.

## 9.2 USB Q6A -> EC25 : VBUS commutable sur la carte Power

D+/D- restent directs afin d'éviter de rallonger inutilement la paire USB2 :

```text
Q6A USB2 host D+ ---------------------------------> EC25 DP
Q6A USB2 host D- ---------------------------------> EC25 DN
Q6A GND ------------------------------------------> EC25 GND
```

Le 5 V VBUS passe désormais obligatoirement par la carte Power :

```text
Q6A USB2 HOST VBUS
        |
        +--> PAD_Q6A_EC25_VBUS_IN
                  |
             source AO3401A / C15127
             drain
                  |
        PAD_EC25_USB_VBUS_OUT
                  |
              EC25 VBUS
```

Commande du switch :

```text
AO3401A gate -> 100 kOhm -> Q6A_EC25_VBUS_IN (source 5 V)
AO3401A gate -> drain AO3400A / C20917
AO3400A source -> GND
GP6 -> 10 kOhm -> gate AO3400A
AO gate -> 1 MOhm -> GND
AO gate -> 1 uF -> GND
```

Le RC de hold protège la continuité USB lors d'un reset MCU court ; un GPIO volontairement LOW coupe rapidement le VBUS via la résistance série 10 kOhm.

```text
GP6 HIGH -> EC25_USB_VBUS ON
GP6 LOW  -> EC25_USB_VBUS OFF
GP6 Hi-Z -> maintien temporaire puis OFF
```

Objectifs :

```text
- pouvoir garantir un vrai hard power-cycle EC25 en coupant BAT + USB_VBUS ;
- pouvoir couper USB_VBUS tout en laissant MODEM_PWR actif si la politique de sleep l'exige ;
- ne pas dépendre du seul comportement de suspend USB du Q6A pour mettre le modem au repos ;
- aucune nouvelle référence Extended : AO3401A/AO3400A/passifs déjà présents dans la BOM.
```

Quectel prévoit explicitement un contrôle de `USB_VBUS` dans son architecture de référence ; si le host ne réalise pas un suspend USB exploitable, VBUS doit pouvoir être retiré pour autoriser le sleep du modem.

## 9.3 Séquences USB/modem

Démarrage recommandé à valider :

```text
MODEM_PWR ON
-> stabilisation BAT carrier
-> pulse EC25 PWRKEY
-> attendre état module/VIO/STATUS exploitable
-> EC25_USB_VBUS ON
-> énumération USB
```

Hard power-cycle :

```text
EC25_USB_VBUS OFF
-> tentative arrêt propre/PWRKEY si possible
-> MODEM_PWR OFF
-> attente décharge rails carrier
-> MODEM_PWR ON
-> PWRKEY
-> EC25_USB_VBUS ON après stabilisation
```

La durée des délais est à caractériser au banc.

Même VBUS coupé, D+/D- restent physiquement connectés ; vérifier que ces lignes n'injectent pas de courant significatif dans un carrier EC25 éteint.

## 9.4 UART / DTR / RI

```text
SN74AVC4T245DR / C22495
VCCA = EC25 VIO (~1.8 V)
VCCB = 3V3_MCU
EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

Les deux directions doivent être placées dans les groupes DIR/OE appropriés du SN74AVC4T245. Prévoir 100 nF sur chaque rail et les pull-up/pull-down prévus par le schéma final. VIO/TXD/RI à mesurer sur le carrier réel.

`RI` peut être utilisé par le MCU comme source d'événement pour réveiller le Q6A via PWR_ON_KEY si la stratégie téléphonie/suspend le nécessite.

## 9.5 PWRKEY / RESET

```text
GP13 -> 10 kOhm -> base S8050 -> EC25 PWRKEY open-collector
base -> 100 kOhm -> GND
```

Recovery graduelle : AT propre -> PWRKEY -> coupure USB_VBUS + MODEM_PWR hard cycle.

GP28 reste réserve 3.3 V + étage RESET_N open-drain DNP optionnel.

---

# 10. Machine d'états Power

## 10.1 États

```text
BAT_DISARMED       : BAT_ARM ouvert ; batterie isolée ; USB-C peut néanmoins alimenter SYS
STORAGE            : MCU hard-off par STORAGE_SW après shutdown
OFF                : MCU ON ; MAIN/MODEM/EC25_USB_VBUS/annexes OFF
BOOT_Q6A           : BQ validé -> MAIN ON -> PWR_ON_KEY -> attente Q6A
RUN_INTERACTIVE    : Android actif, écran généralement ON
RUN_SCREEN_OFF     : Android actif, écran OFF ; musique/appel/GPS/etc. peuvent continuer
DEEP               : Android a décidé lui-même un suspend ; MAIN reste ON
SHUTDOWN_REQUEST   : MCU a demandé un arrêt complet via SHUTDOWN_REQ
SHUTDOWN_PROGRESS  : Android arrête proprement services/eMMC/modem
SHUTDOWN_READY     : Android indique qu'il est sûr de couper MAIN
FAULT              : récupération graduelle ; hard cut en dernier recours
USB_ONLY_MAINT     : USB-C alimente la carte sans batterie exploitable ; pas de boot système lourd par défaut
```

## 10.2 Cold boot

```text
MCU boot
-> GPIO sûrs : MAIN/MODEM/USB_VBUS/ANNEXES OFF
-> BQ initialisé + readback valide
-> MAIN_PWR ON
-> stabilisation J19
-> pulse PWR_ON_KEY
-> attendre HEARTBEAT_STATE RUN
-> si VSYS sûr : MODEM_PWR ON
-> EC25 PWRKEY
-> EC25_USB_VBUS ON après stabilisation modem
```

## 10.3 Appui Power court

```text
bouton utilisateur -> MCU -> pulse PWR_ON_KEY
```

Aucune commande de sleep n'est envoyée par le MCU. Android gère écran, lock, wakelocks et SystemSuspend. Le wake utilisateur utilise le même PWR_ON_KEY.

## 10.4 Extinction complète propre

```text
MCU -> SHUTDOWN_REQ
-> Android : SHUTDOWN_IN_PROGRESS
-> arrêt applications/services + synchronisation stockage
-> arrêt modem propre
-> Android : SHUTDOWN_READY
-> MCU : EC25_USB_VBUS OFF
-> MODEM_PWR OFF
-> MAIN_PWR OFF
-> OFF
```

Le MCU ne coupe pas MAIN sur un simple délai nominal si Android répond encore ; `SHUTDOWN_READY` est l'ACK normal recherché.

## 10.5 Appui long / récupération panne

Ordre de récupération :

```text
1. SHUTDOWN_REQ
2. attendre progrès/ACK
3. si Q6A bloqué : maintien PWR_ON_KEY long, durée à caractériser
4. nouvelle attente
5. si toujours bloqué : EC25_USB_VBUS OFF -> MODEM_PWR OFF -> MAIN_PWR OFF forcé
```

La coupure forcée est le dernier recours car elle peut interrompre des écritures eMMC.

## 10.6 Extinction automatique après longue inactivité

Le MCU ne déduit jamais seul l'inactivité à partir de l'écran ou de l'absence de heartbeat. Android doit explicitement signaler `AUTO_OFF_ALLOWED` via HEARTBEAT_STATE.

```text
AUTO_OFF_ALLOWED
-> MCU démarre/continue un timer configurable
-> si l'autorisation reste valide jusqu'au timeout : SHUTDOWN_REQ
-> extinction propre normale
```

Cette fonction doit être configurable, car un téléphone complètement OFF ne peut plus recevoir appel/SMS.

## 10.7 Reset MCU pendant RUN

```text
RC MAIN/MODEM/EC25_USB_VBUS maintient les rails ~1 s
-> firmware revalide GP10/GP29/GP6 immédiatement
-> resynchronisation BQ/HEARTBEAT/UART
```

---

# 11. SOC / batterie Android

Le MCU utilise la télémétrie BQ : `VBAT / VSYS / IBAT / IBUS / TS / états charge/faults`, avec modèle OCV/SOC adapté JK50. Exposition batterie virtuelle via SBS possible.

Ce n'est pas un fuel-gauge de précision. `MAX17048` non requis en baseline.

---

# 12. Connectique / pastilles THT

La batterie est la seule liaison interne avec connecteur PCB dédié (`Molex 5050060812`). Les autres liaisons sont sur pastilles THT soudées.

Groupes minimum :

```text
BAT_ARM       : BAT_RAW+ / BAT
BAT AUX       : ID1 / ID2 / NTC1 / NTC2 + TPs
Q6A PWR       : MAIN_PWR_OUT / GND
Q6A CTRL      : PWR_ON_KEY / SHUTDOWN_REQ / HEARTBEAT_STATE / SBS_SDA / SBS_SCL / Q6A_3V3_REF / GND
EC25 PWR      : MODEM_PWR_OUT / GND
EC25 USB PWR  : Q6A_EC25_VBUS_IN / EC25_USB_VBUS_OUT / GND
EC25 CTRL     : VIO / TXD / RXD / DTR / RI / PWRKEY / RESET_OD éventuel / GND
ANNEXE1       : ANNEXE1_OUT / GND
ANNEXE2       : ANNEXE2_OUT / GND
BUTTON        : POWER / GND
STORAGE       : 2 pads STORAGE_SW
RESERVE       : GP28 / GND
```

Les pads puissance sont plus gros que les pads signal et doivent avoir un soulagement mécanique externe au pad lorsque possible.

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
EC25_USB_VBUS_IN / EC25_USB_VBUS_GATE / EC25_USB_VBUS_OUT
ANNEXE1_GATE / ANNEXE1_OUT
ANNEXE2_GATE / ANNEXE2_OUT
CC1 / CC2 / CC_SENSE
Q6A_3V3_REF / Q6A_PWRKEY / Q6A_SHUTDOWN_REQ / Q6A_HEARTBEAT_STATE
SBS_SDA / SBS_SCL
EC25_VIO / TXD / RXD / DTR / RI / PWRKEY
TP_GP28_3V3 / TP_EC25_RST_OD
```

---

# 14. PCB / mécanique

```text
PCB : 4 couches
max strict : 70 x 25 mm
RP2040-Tiny : BOTTOM
```

Stack fonctionnel :

```text
L1 : composants + USB2 extérieur + boucles puissance locales
L2 : GND continu
L3 : puissance + signaux lents
L4 : signaux + GND
```

Priorités placement : BQ/boucles -> J_BAT/BAT_ARM -> USB-C/protections -> MAIN/MODEM -> switch EC25 USB VBUS -> annexes -> logique/RP2040.

Le chemin `Q6A_EC25_VBUS_IN -> switch -> EC25_USB_VBUS_OUT` doit être court et dimensionné pour le courant USB du carrier. D+/D- EC25 ne traversent pas la carte et restent en faisceau différentiel court direct.

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
Molex 5050060812   C779875
```

Aucune nouvelle référence Extended n'est ajoutée par la commutation USB_VBUS EC25.

## 15.2 Basic principaux

```text
AO3400A        C20917   drivers MAIN/MODEM + ANNEXE1/2 + EC25_USB_VBUS
AO3401A        C15127   ANNEXE1 + ANNEXE2 + EC25_USB_VBUS
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

Quantités finales à générer depuis le schéma correspondant à cette V0.8.

---

# 16. Fonctions explicitement absentes / interdites en V1

```text
USB-C host / DRP / contrôleur CC         absent
USB-PD / 9 V                             absent
GPIO séparé WAKE Q6A                     absent ; PWR_ON_KEY fait Power + wake
SLEEP_REQ forcé                          supprimé ; remplacé par SHUTDOWN_REQ
BQ_INT vers RP2040                       absent ; polling I2C
Volume +/- via RP2040                    absent ; Q6A directs
EC25 puissance principale via USB_VBUS   interdite ; BAT/MODEM_PWR reste l'énergie principale
USB EC25 D+/D- à travers PCB Power       absent ; données directes Q6A<->EC25
USB_VBUS EC25 non commutable             supprimé : VBUS doit traverser le switch de la carte Power
EC25 RESET dédié                         absent ; option DNP GP28
DISPLAY_PWR sur carte Power              absent
codec/jack/mux audio sur carte Power     absent
CMC USB2 extérieur                       absent baseline
connecteurs Micro-Fit/JST de faisceau    absents ; pads THT
2e BMS/PCM batterie                      absent
fusible batterie obligatoire             absent baseline
Schottky/P-MOS anti-inversion BAT        absent baseline
```

---

# 17. Gates BLOQUANTES avant fabrication

## Batterie / BQ

```text
[ ] vraie JK50 Genuine Service Pack reçue et mesurée
[ ] pinout et mating Molex vérifiés physiquement
[ ] NTC1/NTC2 caractérisés ; RT1/RT2 figés
[ ] BAT_ARM >=5 A, faible R, pads/cuivre adaptés
[ ] budget courant JK50 pire cas validé
[ ] séquence BQ cold boot écrite : watchdog, config, readback, fail-safe
[ ] comportement BQ sans batterie / USB-only caractérisé
```

## Q6A / power

```text
[ ] J19 mods : R7 DNP ; R24 2 mOhm ; R190 100 k ; R191 10 k ; FB4 DNP ; R185..R189 confirmés
[ ] J19 : 4.2 / 3.8 / 3.4 / 3.0 V boot/idle/charge/deep
[ ] PWR_ON_KEY : cold boot, appui court Android, wake, maintien long récupération
[ ] SHUTDOWN_REQ GPIO58 + daemon Android validés
[ ] HEARTBEAT_STATE GPIO59 : RUN / shutdown progress / shutdown ready / auto-off allowed
[ ] shutdown propre confirmé avant coupure MAIN
[ ] MAIN OFF + USB-C présent : consommation/rails Q6A acceptables et reboot propre
[ ] reset MCU court : MAIN ne chute pas
[ ] SBS sans back-power
```

## EC25 / USB

```text
[ ] EC25 VIO/TXD/RI mesurés sur carrier réel
[ ] SN74AVC4T245 directions/OE corrects
[ ] EC25 BAT toujours dans plage sûre avec baseline SYS direct
[ ] switch Q6A->EC25 USB_VBUS fonctionne, OFF par défaut, hold reset MCU correct
[ ] MODEM_PWR OFF + EC25_USB_VBUS OFF : vrai hard-off du carrier/modem
[ ] avec BAT modem OFF, vérifier que D+/D- seuls ne back-powerent pas significativement le carrier
[ ] séquence MODEM_PWR/PWRKEY/USB_VBUS et réénumération USB validées
[ ] suspend EC25 : d'abord tester USB suspend Q6A ; sinon couper VBUS avec GP6 tout en gardant MODEM_PWR ON
[ ] EC25 RI -> politique wake Q6A si requise pour appels/SMS
[ ] pire cas LTE TX + Q6A charge + batterie faible : aucun reset/chute dangereuse
```

## Annexes / PCB

```text
[ ] drivers AO3401A/AO3400A ANNEXE1/2 vérifiés : OFF garanti avec MCU reset/Hi-Z
[ ] ANNEXE1 ampli ON/OFF sans pop/reboot excessif
[ ] ANNEXE2 charge aval compatible SYS
[ ] TPS610995 shutdown/isolation/courant storage validés
[ ] footprint RP2040-Tiny 1:1 + FPC accessible
[ ] layout BQ comparé au layout TI
[ ] USB-C D+/D- recalculé sur stack-up JLC
[ ] ERC/DRC/BOM/PnP/polarités/Gerbers revus
[ ] PCB <=70 x 25 mm sans collision TOP/BOTTOM
```

---

# 18. Tests après fabrication

```text
BAT_ARM ouvert/fermé, USB présent/absent
USB-C 5 V -> BQ -> SYS sans batterie
batterie seule -> SYS
plug/unplug USB sans reboot
BQ : ILIM/IINDPM/VREG/VSYSMIN/ICHG/TS/ADC/watchdog/readback
RP stable + Storage OFF courant résiduel
MAIN/MODEM/EC25_USB_VBUS OFF par défaut
reset MCU court : rails maintenus selon hold
Q6A PWR_ON_KEY court/wake/long
SHUTDOWN_REQ + SHUTDOWN_READY + hard-cut fallback
AUTO_OFF_ALLOWED / annulation timer
VOL+ / VOL- Android natifs
USB-C ADB/EDL/MTP
Q6A MAIN OFF + USB-C présent : état électrique stable
EC25 BAT sûr
EC25 USB : VBUS switch + énumération + coupure + réénumération
EC25 MODEM_PWR OFF + VBUS OFF : hard-off réel
EC25 D+/D- connectés avec modem off : pas de back-power significatif
EC25 sleep avec USB suspend puis scénario VBUS coupé
EC25 UART/DTR/RI/PWRKEY
ANNEXE1/2 ON/OFF
consommations RUN / écran OFF / deep / OFF / storage-off
```

---

# 19. Sources primaires

- Texas Instruments — BQ25628E datasheet / NVDC power-path.
- Texas Instruments — TPS61099x documentation.
- Texas Instruments — SN74AVC4T245 documentation.
- Quectel — EC25 Series Hardware Design V2.4 + EC25 Reference Design (contrôle USB_VBUS).
- Radxa — Dragon Q6A V1.21 schematic + GPIO/USB documentation.
- Waveshare — RP2040-Tiny schematic/mécanique.
- Motorola repair schematics utilisant JK50.
- Molex — série 505006/505004, drawing officiel.
- JLCPCB/LCSC — bibliothèque et règles de fabrication.

---

# 20. Points d'architecture courants

```text
VALIDÉS / NORMATIFS
- JK50 + connecteur Molex + BAT_ARM
- BQ25628E, polling I2C, initialisation/readback obligatoire avant cold boot Q6A
- MAIN_PWR et MODEM_PWR séparés, un JMTQ55P02A chacun, hold RC
- ANNEXE1/2 : AO3401A pilotés par AO3400A, OFF déterministe, aucun hold
- bouton Power court -> PWR_ON_KEY ; Android seul maître du suspend normal
- GP9 = SHUTDOWN_REQ ; aucun SLEEP_REQ forcé
- GPIO59 = HEARTBEAT_STATE incluant SHUTDOWN_READY / AUTO_OFF_ALLOWED
- extinction progressive : shutdown propre -> long PWR_ON_KEY -> hard cut en dernier recours
- USB EC25 : D+/D- directs, VBUS 5 V obligatoirement routé via carte Power et commutable
- GP6 = EC25_USB_VBUS_EN avec AO3401A/AO3400A et hold RC
- hard power-cycle modem = USB_VBUS OFF + MODEM_PWR OFF
- pas de double MOS back-to-back MAIN/MODEM baseline ; validation physique avant complexification
- VOL+/VOL- directs Q6A
- GP28 réserve + RESET_N EC25 open-drain DNP

OUVERT / À CARACTÉRISER
- tension finale de charge JK50 et éventuelle nouvelle architecture d'alimentation principale EC25
- NTC JK50 / valeurs TS finales
- seuils batterie faible et politique power-limit
- timings exacts PWR_ON_KEY / SHUTDOWN_REQ / hard recovery
- codage temporel exact HEARTBEAT_STATE
- comportement Q6A MAIN OFF + USB-C présent
- comportement carrier EC25 avec BAT/VBUS/D+/D- dans tous les états
- politique TREG / ITERM / IBAT_PK après essais thermiques et courants
```

Toute modification ultérieure d'un élément normatif doit être traitée comme une révision d'architecture ou une ECO explicite, jamais comme une correction silencieuse du schéma.