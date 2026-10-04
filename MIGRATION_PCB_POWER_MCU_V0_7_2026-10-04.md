# Migration schéma / PCB — Power MCU V0.7

**Date :** 2026-10-04  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Référence normative :** `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md`  
**Objet :** traduire le CDC V0.7 gelé en modifications KiCad concrètes : schéma, netlist, placement, routage, testpoints, BOM et contrôles.

Ce document n'est pas un CDC parallèle. En cas de contradiction, le CDC V0.7 prévaut.

---

# 1. Migration batterie — priorité n°1

## 1.1 Remplacer l'ancienne interface batterie

Supprimer de la baseline les anciens connecteurs batterie Micro-Fit/JST/THT 3 fils lorsque présents.

Ajouter :

```text
J_BAT = Molex 5050060812
LCSC  = C779875
mate batterie = Molex 5050040812
batterie = Motorola JK50
```

Pinout :

```text
1 -> GND / BATT-
2 -> BAT_ID2
3 -> BAT_ID1
4 -> BAT_RAW+
5 -> BAT_RAW+
6 -> BAT_NTC1
7 -> BAT_NTC2
8 -> GND / BATT-
```

Pins 4+5 sont réunies immédiatement par cuivre large. Pins 1+8 sont raccordées au plan GND avec vias multiples et chemin très court.

Le footprint doit être construit/contrôlé depuis le drawing Molex officiel. Pas de footprint déduit d'une photo. Vérification 1:1 obligatoire avant fabrication.

## 1.2 BAT_ARM

Créer deux nets distincts :

```text
BAT_RAW = côté batterie avant interrupteur
BAT     = côté carte/BQ après interrupteur
```

Chaîne :

```text
J_BAT 4+5 -> BAT_RAW
BAT_RAW -> PAD_BAT_ARM_A
PAD_BAT_ARM_A -> interrupteur SPST externe >=5 A
interrupteur -> PAD_BAT_ARM_B
PAD_BAT_ARM_B -> BAT
BAT -> pin BAT BQ25628E
```

Les deux pads BAT_ARM sont THT, gros, accessibles sur bord de PCB, avec sérigraphie explicite `BAT_RAW` et `BAT`.

Ajouter TP_BAT_RAW et TP_BAT.

Aucun fusible, PTC, Schottky ou P-MOS anti-inversion obligatoire en série dans la baseline V0.7. Si un ancien `F_BAT/SJ_BAT` occupe une zone utile, le retirer. Un footprint DNP ne peut rester que s'il ne dégrade ni le routage ni la surface.

## 1.3 NTC / ID

Créer :

```text
TP_BAT_ID1
TP_BAT_ID2
TP_BAT_NTC1
TP_BAT_NTC2
```

ID1/ID2 restent en haute impédance, sans pull-up imposée.

Prévoir deux sélections exclusives vers BQ_TS :

```text
BAT_NTC1 -> RSEL_NTC1 0R/DNP -> NTC_SEL
BAT_NTC2 -> RSEL_NTC2 0R/DNP -> NTC_SEL
NTC_SEL -> réseau TS BQ25628E
```

Une seule résistance de sélection sera montée.

Ajouter si la place le permet :

```text
TP_NTC_EXT
PAD_NTC_EXT
GND adjacent
```

pour un NTC externe de secours.

Les valeurs RT1/RT2 historiques 5.1 k / 30 k restent des footprints de départ mais ne sont pas figées tant que la JK50 réelle n'a pas été caractérisée.

---

# 2. BQ25628E

## 2.1 Supprimer BQ_INT vers MCU

Supprimer :

```text
BQ_INT -> GP6
pull-up BQ_INT -> 3V3_MCU
```

Le pin INT peut rester NC. Un TP facultatif est acceptable si gratuit.

GP6 devient :

```text
GP6_RESERVE -> pastille THT + TP
```

Aucune autre charge sur GP6.

## 2.2 Conserver la configuration chargeur

Schéma cible :

```text
CE -> GND
SDA -> GP4 + 10 k vers 3V3_MCU
SCL -> GP5 + 10 k vers 3V3_MCU
RILIM = 5.6 k
PG -> TP
STAT -> NC
QON -> TP si prévu
```

VREG 4.15 V et ICHG <=2 A sont des paramètres firmware ; ne pas créer de réseau matériel contradictoire.

## 2.3 Placement/routage BQ

Le BQ est la priorité absolue du placement. Refaire le placement si nécessaire pour obtenir :

```text
PMID caps collés à PMID/GND
SYS caps collés à SYS
CVBUS/CBAT/REGN au plus près
SW très court et très compact
inductance collée au switcher
vias thermiques/GND abondants
aucun signal sensible sous/près du nœud SW
```

Ne pas préserver un ancien placement uniquement pour conserver des pistes déjà routées.

---

# 3. Arbre de puissance et classes de nets

Créer/contrôler les nets de puissance :

```text
BAT_RAW
BAT
SYS
MAIN_PWR
MODEM_PWR
ANNEXE1_OUT
ANNEXE2_OUT
MCU_VSYS
VBUS_USB_C
```

## 3.1 BAT_RAW / BAT / SYS

Le courant pack documenté JK50 approche 4.85 A. Les chemins principaux doivent donc être conçus comme des **zones/pours de puissance**, pas comme de longues pistes fines.

Règle de routage recommandée :

```text
BAT_RAW/BAT/SYS : cuivre large/pour sur couche externe, objectif >=2.5 mm si piste contrainte,
                  préférer nettement plus large dès que la géométrie le permet.
MAIN_PWR        : >=1.5 mm si piste, de préférence zone/pour.
MODEM_PWR       : >=1.5 mm si piste, de préférence zone/pour.
ANNEXE1/2       : >=0.8 mm pour cible ~1 A, à adapter à la longueur.
logique lente   : 0.20–0.25 mm typique.
```

Ces valeurs sont des minimums de travail, pas un substitut au calcul final avec le stack-up JLC et le cuivre réel.

Sur les changements de couche puissance : utiliser plusieurs vias en parallèle. Éviter de faire passer le courant batterie principal par un unique via.

Le trunk SYS doit partir du bloc BQ vers un point de distribution court puis se séparer vers MAIN/MODEM/MCU/annexes. Éviter un chaînage série où le courant modem traverse la branche Q6A ou inversement.

## 3.2 GND

L2 reste un plan GND continu. Ne pas le découper pour faire passer des pistes.

Les retours de puissance BQ, Q6A, EC25 et batterie doivent rejoindre un plan GND à faible impédance avec vias multiples. Les retours USB et signaux sensibles gardent un plan de référence continu.

---

# 4. MAIN_PWR / MODEM_PWR

Conserver pour chacun :

```text
JMTQ55P02A high-side
AO3400A driver
P-gate -> 100 k -> SYS
GPIO -> 10 k -> AO gate
AO gate -> 1 M -> GND
AO gate -> 1 uF -> GND
```

Nets :

```text
GP10 -> MAIN_PWR_EN
GP29 -> MODEM_PWR_EN
```

Le 1 uF doit être proche de l'AO3400A concerné. Le chemin grille du P-MOS doit rester court.

Ne jamais placer le condensateur de hold directement sur la grille du P-MOS de puissance.

MAIN_PWR et MODEM_PWR doivent rester des branches séparées physiquement après SYS.

---

# 5. ANNEXE1 / ANNEXE2

Conserver deux rails séparés, OFF par défaut, sans RC hold :

```text
GP11 -> ANNEXE1_EN -> AO3401A -> ANNEXE1_OUT
GP12 -> ANNEXE2_EN -> AO3401A -> ANNEXE2_OUT
```

Sérigraphie/fonctions :

```text
ANNEXE1 = AMP / ampli haut-parleur
ANNEXE2 = AUX / caméras, flash, outils, futurs périphériques
```

ANNEXE2 reste SYS brut commuté. Ne pas sérigraphier `5V` ou `3V3`.

Sorties par grosses pastilles THT + GND adjacent.

---

# 6. Connectique hors batterie

Supprimer les anciens Micro-Fit/JST pour Q6A/EC25/annexes lorsqu'ils existent.

Créer des groupes THT clairement séparés :

```text
Q6A_PWR : MAIN_PWR / GND
Q6A_CTRL: PWR_ON_KEY / SLEEP_REQ / HEARTBEAT / SBS_SDA / SBS_SCL / Q6A_3V3 / GND
EC25_PWR: MODEM_PWR / GND
EC25_CTRL: VIO / TXD / RXD / DTR / RI / PWRKEY / RESET_OD option / GND
ANNEXE1 : OUT / GND
ANNEXE2 : OUT / GND
POWER_BUTTON : GP7 / GND
STORAGE_SW : 2 pads
BAT_ARM : 2 pads puissance
RESERVE : GP6/GND et GP28/GND
```

Pads puissance plus gros que pads logiques. Les groupes doivent rester accessibles au fer après assemblage du RP2040 et montage mécanique.

---

# 7. RP2040-Tiny

Pinout PCB à contrôler intégralement :

```text
GP0   EC25 RXD
GP1   EC25 TXD
GP2   EC25 DTR
GP3   EC25 RI
GP4   BQ SDA
GP5   BQ SCL
GP6   RESERVE
GP7   PWR_BUTTON
GP8   Q6A PWR_ON_KEY
GP9   Q6A SLEEP_REQ
GP10  MAIN_PWR_EN
GP11  ANNEXE1_EN
GP12  ANNEXE2_EN
GP13  EC25 PWRKEY
GP14  SBS_SDA
GP15  SBS_SCL
GP26  CC_SENSE ADC
GP27  HEARTBEAT
GP28  RESERVE / RESET_N OD DNP
GP29  MODEM_PWR_EN
```

GP6 ne doit plus comporter de piste vers le BQ.

GP28 conserve :

```text
TP_GP28_3V3
+ étage S8050 RESET_N entièrement DNP
```

Le RP2040-Tiny reste BOTTOM avec keepout sous module et FPC accessible.

---

# 8. Q6A

Conserver les lignes vers la carte :

```text
GP8 -> S8050 -> Q6A PWR_ON_KEY
GP9 -> GPIO58 SLEEP_REQ
GPIO59 -> GP27 HEARTBEAT
GP14/15 <-> I2C6 SBS
Q6A_3V3_REF
MAIN_PWR -> J19
GND
```

SBS : deux 4.7 k vers Q6A_3V3 DNP, jamais vers 3V3_MCU.

Les boutons Volume ne sont pas routés au RP2040 ni à un ADC :

```text
VOL+ -> Q6A J20 pin 29 / GPIO31
VOL- -> Q6A J20 pin 32 / GPIO30
```

Ils peuvent être câblés directement hors carte Power. Ne pas consommer GP6/GP28 pour eux.

---

# 9. EC25

Conserver :

```text
SYS -> MODEM_PWR -> carrier BAT
Q6A USB2 host -> EC25 USB direct hors PCB
```

Le PCB Power ne doit pas transporter D+/D- du lien Q6A-EC25.

SN74AVC4T245 :

```text
VCCA = EC25 VIO
VCCB = 3V3_MCU
EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

PWRKEY par S8050 sur GP13.

RESET_N uniquement via étage DNP GP28.

Placer les pads EC25 CTRL près du bord et tenir VIO/UART éloignés du nœud SW du BQ.

---

# 10. USB-C externe

Conserver USB-C device/sink uniquement :

```text
CC1 -> 5.1 k -> GND
CC2 -> 5.1 k -> GND
CC1 --470k--+
            +-> CC_SENSE GP26
CC2 --470k--+
CC_SENSE -> 100 nF -> GND
```

D+/D- vont au Q6A OTG via ESD. VBUS va au BQ et au Q6A OTG via B5819W.

Placement :

```text
USB-C au bord
TVS VBUS près du connecteur
ESD USB/CC près du connecteur
D+/D- appairées et courtes
aucun stub
plan GND L2 continu sous la paire
```

Recalculer largeur/espacement USB2 pour ~90 ohm différentiel avec le stack-up réellement commandé chez JLCPCB.

Éloigner la paire USB du nœud SW et de l'inductance BQ.

---

# 11. Placement PCB V0.7

Ordre de priorité :

```text
1. BQ25628E + passifs + inductance
2. J_BAT + BAT_ARM + chemin BAT
3. USB-C + protections
4. MAIN_PWR / MODEM_PWR
5. TPS610995
6. ANNEXE1 / ANNEXE2
7. level shifter EC25 / transistors PWRKEY
8. RP2040-Tiny et signaux lents
9. groupes THT/testpoints
```

J_BAT doit être orienté pour que le flex JK50 arrive sans pli serré et sans traverser le BQ/USB-C.

La contrainte PCB reste <=70 x 25 mm. Si J_BAT impose un conflit mécanique, déplacer les blocs non critiques avant de dégrader le placement BQ.

---

# 12. Testpoints V0.7

Minimum :

```text
VBUS_USB_C
BQ_VBUS
BAT_RAW
BAT
SYS
GND
BAT_ID1 / ID2
BAT_NTC1 / NTC2
BQ_TS
BQ_SDA / SCL / ILIM / PG
MCU_VSYS / 3V3_MCU / TPS_EN
MAIN_GATE / MAIN_PWR
MODEM_GATE / MODEM_PWR
ANNEXE1 / ANNEXE2
CC1 / CC2 / CC_SENSE
Q6A_3V3 / PWRKEY / SLEEP_REQ / HEARTBEAT
SBS_SDA / SBS_SCL
EC25_VIO / TXD / RXD / DTR / RI / PWRKEY
GP6_RESERVE
GP28_3V3 / EC25_RESET_OD
```

---

# 13. Net classes recommandées

Créer au minimum :

```text
PWR_BAT       : BAT_RAW, BAT, SYS
PWR_HIGH      : MAIN_PWR, MODEM_PWR
PWR_AUX       : ANNEXE1_OUT, ANNEXE2_OUT, MCU_VSYS
USB2_DIFF     : USB D+/D-
ANALOG_SENSE  : CC_SENSE, BQ_TS, ID/NTC
LOGIC         : GPIO/I2C/UART
```

ANALOG_SENSE doit éviter le SW/inductance et ne pas partager de longs parcours parallèles avec les pistes de puissance commutée.

---

# 14. Contrôles avant DRC final

Schéma :

```text
[ ] aucune référence Micro-Fit/JST obsolète hors J_BAT Molex
[ ] BQ_INT déconnecté de GP6
[ ] GP6 réserve correctement exposée
[ ] J_BAT pinout exact 1/8 GND, 4/5 BAT+, 2/3 ID, 6/7 NTC
[ ] BAT_RAW et BAT distincts, BAT_ARM entre les deux
[ ] sélecteurs NTC1/NTC2 exclusifs
[ ] aucun deuxième BMS/fusible/anti-inversion imposé
[ ] pinout RP conforme CDC V0.7
[ ] ANNEXE1/2 nommées selon usage
```

PCB :

```text
[ ] footprint J_BAT vérifié 1:1 et selon drawing Molex
[ ] BAT/SYS routés en cuivre large avec vias multiples
[ ] BQ comparé visuellement au layout TI
[ ] aucun découpage de plan GND sous USB
[ ] USB2 calculée sur stack-up réel
[ ] SW court/compact et éloigné des sense
[ ] FPC RP2040 accessible
[ ] tous les THT accessibles après assemblage
[ ] PCB <=70 x 25 mm
[ ] sérigraphie BAT_RAW/BAT/GND explicite
```

Fabrication :

```text
[ ] ERC propre
[ ] DRC propre
[ ] BOM régénérée
[ ] PnP contrôlé
[ ] polarités/orientations contrôlées manuellement
[ ] C779875 stock JLC/LCSC revalidé
[ ] Gerbers visualisés couche par couche
```

---

# 15. Critère de fin de migration

La migration KiCad V0.7 est terminée uniquement lorsque :

```text
1. le schéma implémente le CDC V0.7 sans contradiction ;
2. le PCB est resynchronisé depuis le schéma ;
3. le placement puissance a été revu, pas seulement les ratsnests raccordés ;
4. BAT_RAW/BAT/SYS et MAIN/MODEM ont été reroutés pour leur courant réel ;
5. la paire USB2 est recalculée/reroutée si nécessaire ;
6. ERC/DRC sont propres ou chaque exception est documentée ;
7. BOM/PnP/Gerbers correspondent à cette révision ;
8. aucune décision historique supprimée ne subsiste silencieusement dans le PCB.
```

Le fichier KiCad de travail doit ensuite être versionné comme source V0.7 afin que les prochaines revues portent sur le projet réel et non sur un ZIP historique.