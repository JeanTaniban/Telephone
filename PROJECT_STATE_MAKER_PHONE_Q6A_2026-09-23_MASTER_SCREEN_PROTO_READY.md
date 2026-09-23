# PROJECT STATE — Maker Phone / Radxa Dragon Q6A — écran V1 prototype

**Date :** 2026-09-23  
**Statut :** addendum maître pour l’interposer écran V1, prêt à passer au PCB après correction J2 ci-dessous.  
**Base :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-22_MASTER_ANDROID_BOOT_OK.md`.  
**Règle de priorité :** pour tout ce qui concerne l’écran, son connecteur, le pinout J2, le backlight et l’interposer V1, **ce document supersède les sections 5, 11, 12, 21 et 24 du master du 2026-09-22 lorsqu’elles sont contradictoires**. Toutes les autres sections du master du 2026-09-22 restent applicables.

---

# 1. Matériel écran réellement reçu

[CONFIRMÉ PAR INSPECTION]

```text
marquage module : FMS780 JB2-1
connecteur écran : mâle / header
contacts électriques : 40
organisation mécanique : 2 rangées de 20
marquage visible sur flex : 1 / 20 / 21 / 40
```

Le marquage `1 / 20 / 21 / 40` du flex **ne doit pas être utilisé comme preuve de la numérotation électrique Panasonic**. Il est conservé seulement comme repère mécanique du fabricant du flex.

Observation réelle de continuité, écran hors tension :

```text
extrémité du flex marquée 40 / 1  : continuité entre les deux contacts d’extrémité
extrémité du flex marquée 21 / 20 : pas de continuité entre les deux contacts d’extrémité
```

Cette observation est cohérente avec l’hypothèse retenue pour la V1 : l’extrémité électrique `39 / 40` est GND/GND, tandis que l’extrémité électrique `1 / 2` est NC/IOVCC. Ce contrôle augmente fortement la confiance dans le sens retenu, sans constituer une identification métrologique contact par contact.

---

# 2. Connecteur J2 retenu sur l’interposer

[CIBLE V1 GELÉE]

```text
J2 : Panasonic AXE540127
socket / femelle
40 contacts
pitch 0,4 mm
LCSC C3652918
footprint : interposer_q6a:Panasonic_AXE540127_40P_P0.4mm
```

Le connecteur monté sur l’écran est mâle ; l’interposer doit donc porter le socket femelle AXE540127.

Risque résiduel accepté pour la V1 : le mating exact avec cet écran aftermarket n’a pas encore été validé par emboîtement physique d’un AXE540127 réel. Le nombre de contacts, le sexe, le pitch et l’encombrement sont cohérents.

Les quatre pads mécaniques/blindage `G1…G4` / pads footprint `41…44` restent raccordés à GND/châssis comme prévu. Ils ne font pas partie des 40 contacts fonctionnels du panneau.

---

# 3. Pinout électrique Motorola à utiliser comme source de vérité

Source primaire : schéma Motorola Yukon / Moto G200, page 86/90, feuille `DISP: CONNECTOR`, connecteur `J1891`.

## 3.1 Correction obligatoire par rapport à AdaptateurEcran(5)

Le schéma KiCad actuel contient encore l’inversion suivante :

```text
ACTUEL / FAUX
J2.1 = DISP_IOVCC_1P8
J2.2 = NC
```

À remplacer par :

```text
CIBLE / CORRECT
J2.1 = NC
J2.2 = DISP_IOVCC_1P8
```

La pin 1 doit recevoir un marqueur `No Connect`. La pin 2 doit être reliée au rail `DISP_IOVCC_1P8`.

## 3.2 Pinout V1 gelé

```text
 1  NC
 2  DISP_IOVCC_1P8
 3  DISP_VSP              +5,5 V
 4  DISP_RST_N
 5  DISP_VSN              -5,7 V
 6  DISP_TE
 7  NC
 8  DISP_PWM_OUT
 9  NC
10  GND
11  WLED_A
12  DSI_D2_P
13  WLED_K1
14  DSI_D2_N
15  WLED_K2
16  GND
17  WLED_K3
18  DSI_D1_P
19  NC
20  DSI_D1_N
21  TP_INT_N
22  GND
23  NC
24  DSI_CLK_P
25  NC
26  DSI_CLK_N
27  TP_RST_N
28  GND
29  GND
30  DSI_D0_P
31  TP_SPI_CS_N
32  DSI_D0_N
33  TP_SPI_MISO
34  GND
35  TP_SPI_MOSI
36  DSI_D3_P
37  TP_SPI_CLK
38  DSI_D3_N
39  GND
40  GND
```

Masses fonctionnelles du panneau :

```text
10, 16, 22, 28, 29, 34, 39, 40
```

L’asymétrie des masses est volontairement utilisée comme repère de sens mécanique.

---

# 4. Marquage d’orientation obligatoire sur le PCB

[OBLIGATOIRE AVANT FAB]

Le connecteur J2 doit être orientable sans avoir à interpréter le marquage ambigu du flex.

Le footprint actuel numérote :

```text
côté X négatif : pads 1 / 2
côté X positif : pads 39 / 40
```

Le footprint possède déjà un petit marqueur standard de pin 1 côté `1 / 2`. **Le conserver.**

Ajouter en plus un repère d’orientation utilisateur, non ambigu, du côté `39 / 40`, qui est l’extrémité GND/GND :

```text
● 39/40 GND
```

Exigences :

- point plein sérigraphié visible, diamètre cible ~1 mm ;
- placé à l’extrémité `39 / 40`, sans empiéter sur pads, courtyard ou zone de mating ;
- ajouter le texte court `39/40 GND` ou `GND END` sur `F.SilkS` si la place le permet ;
- reproduire le repère sur `F.Fab` pour qu’il reste explicite dans les exports de fabrication ;
- ce point **n’est pas un marqueur pin 1** : le marqueur pin 1 existant reste à l’autre extrémité.

Sur la carte assemblée, le côté du flex où l’on a observé la continuité entre les deux contacts d’extrémité doit se retrouver du côté du repère `● 39/40 GND`.

---

# 5. DSI

[CIBLE V1 INCHANGÉE]

```text
MIPI DSI D-PHY
4 lanes + clock
liaison directe Q6A -> panneau
pas de TPD4E05U06 sur l’interposer interne
routage 100 ohms différentiel au PCB
60 Hz au premier bring-up
DSC activé selon configuration Moto
```

Aucune modification fonctionnelle DSI n’est demandée par cet addendum, hormis la correction de J2.1/J2.2 qui concerne IOVCC.

---

# 6. Bias LCD

[CIBLE V1 INCHANGÉE]

```text
TPS65132WRVCT
VIN = LCD_3V3 Q6A
VSP cible = +5,5 V
VSN cible = -5,7 V
ENP et ENN séparés
pull-down 100 kΩ sur ENP et ENN
pads filaires THT pour LCD_3V3 / ENP / ENN
pads de mesure VSP / VSN
programmation NVM par J3, adaptateur isolé du Q6A et de l’écran
```

Aucune dalle ne doit être connectée pendant la programmation/validation initiale du bias.

---

# 7. Tactile

[CIBLE V1 INCHANGÉE]

Le chemin principal reste le tactile Novatek SPI 1,8 V via 4×74LVC2T45 vers le SPI6 Q6A.

Le breakout passif J4 expose en plus le tactile natif J10 Q6A :

```text
TP_RESET
TP_VCC 3,3 V
TP_INT
TP_SDA
TP_SCL
GND
```

Ce breakout n’est pas un pont I²C/SPI et ne remplace pas le chemin tactile SPI principal.

---

# 8. Backlight — décision révisée et désormais cible V1

La décision historique « utiliser le SY7203 natif Q6A comme chemin principal et modifier R315/C644 » est **supplantée**.

[CIBLE V1]

Le backlight principal est désormais le **TPS61194PWPR intégré à l’interposer**.

```text
U2 TPS61194PWPR
L2 4,7 µH
D1 Schottky 60 V / 3 A
3 × Cin 10 µF / 25 V
3 × Cout 10 µF / 100 V
RFSET 27,4 kΩ
RISET 105 kΩ
courant cible ~24,8 mA/string
VBOOST_MAX ~30,9 V
OUT1 -> K1
OUT2 -> K2
OUT3 -> K3
OUT4 -> traitement conforme datasheet, actuellement GND
```

Les trois strings `K1/K2/K3` restent indépendantes.

## 8.1 Source BL_VIN

Deux sources sélectionnables, toutes deux ouvertes par défaut :

```text
Q6A_5V_IN  -> 0 Ω DNI -> BL_VIN
BL_VIN_EXT -> 0 Ω DNI -> BL_VIN
```

Règle absolue : **une seule source BL_VIN à la fois**.

`Q6A_5V_IN` est un fil vers `VCC_5V_PERI` du Q6A ; ce 5 V ne vient pas de J10.

`BL_VIN_EXT + GND` reste exposé pour alimentation de laboratoire ou future source système.

## 8.2 Chemin backlight natif Q6A

Le SY7203 natif est conservé seulement comme voie expérimentale/fallback. Il est isolé du panneau par composants DNI.

Le mode normal TPS61194 ne nécessite **aucune modification de R315 ni de C644**.

Mesures réelles C644 effectuées :

```text
capacité in-circuit : ~2,2 µF confirmée
dimensions approximatives : ~1,5 × 1 mm
tension nominale : non déterminable au multimètre
```

Le schéma Q6A indique 10 V, mais ce rating n’est plus critique pour le chemin TPS61194 puisque le SY7203 natif n’est pas le chemin normal.

---

# 9. État de validation du schéma V1

Après correction J2.1/J2.2 et ajout du repère d’orientation `● 39/40 GND`, le schéma est considéré **gelable pour prototype V1**, sous réserve des contrôles KiCad suivants :

```text
[ ] J2.1 = NC
[ ] J2.2 = DISP_IOVCC_1P8
[ ] aucun ancien label/commentaire ne dit pin 1 = IOVCC
[ ] footprint J2 inchangé électriquement, AXE540127 femelle
[ ] point d’orientation visible côté pads 39/40
[ ] marqueur pin 1 standard conservé côté pads 1/2
[ ] pads 41-44 / G1-G4 raccordés GND/châssis
[ ] TPS61194 reste voie normale
[ ] sélecteurs BL_VIN DNI par défaut
[ ] chemin SY7203 natif isolé par DNI
[ ] ERC KiCad = 0 erreur / 0 avertissement
[ ] export netlist contrôlé sur J2 pins 1,2,10,16,22,28,29,34,39,40
[ ] export netlist contrôlé sur WLED_A/K1/K2/K3
[ ] BOM ne change pas pour J2 ; références TPS61194/backlight cohérentes
[ ] journal projet mis à jour
```

Le dernier ERC documenté dans `AdaptateurEcran(5)` est antérieur à la correction J2.1/J2.2 ; il doit donc être relancé après la modification.

---

# 10. Risque résiduel accepté pour la première PCB

Il reste impossible de sonder proprement l’ensemble des 40 contacts du connecteur mâle écran sans breakout dédié.

La V1 s’appuie donc sur quatre éléments convergents :

```text
1. schéma Motorola J1891 pour le pinout électrique
2. footprint/datasheet Panasonic pour la numérotation du socket
3. inspection réelle : écran = connecteur mâle 40 contacts
4. continuité réelle asymétrique : côté flex 40/1 commun, côté 21/20 non commun
```

Le PCB doit être conçu pour rendre une erreur de sens détectable avant mise sous tension : repère `● 39/40 GND`, testpoints accessibles et bring-up avec backlight désactivé.

---

# 11. Séquence de bring-up écran après fabrication

```text
1. inspection optique connecteur J2 et orientation du repère ● 39/40 GND
2. aucun écran : vérifier courts-circuits / rails / EN
3. valider TPS65132 seul : +5,5 V / -5,7 V
4. valider TPS61194 sans écran, BL_VIN limité en courant
5. brancher écran avec backlight OFF
6. vérifier IOVCC 1,8 V, VSP, VSN
7. DSI 60 Hz + reset + DCS
8. seulement ensuite activer backlight à faible PWM
9. contrôler courant K1/K2/K3 et température driver/diode/inductance
10. tactile SPI en dernier après stabilité display
```

Aucun branchement/débranchement écran à chaud.

---

# 12. Décisions explicitement supplantées par cet addendum

| Ancienne décision | État V1 2026-09-23 |
|---|---|
| J2.1 = IOVCC / J2.2 = NC | **FAUX : J2.1 = NC, J2.2 = IOVCC 1,8 V** |
| Se fier directement au marquage flex 1/20/21/40 | **NON : repère mécanique uniquement** |
| SY7203 Q6A = backlight principal | **SUPPLANTÉ : TPS61194 interposer = chemin principal** |
| Retirer R315 avant fonctionnement normal | **NON requis en mode TPS61194** |
| Remplacer C644 pour fonctionnement normal | **NON requis en mode TPS61194** |
| 3×47 Ω = chemin normal backlight | **fallback natif Q6A uniquement, DNI par défaut** |
| RSET 8,2 Ω = réglage normal | **historique fallback ; TPS61194 utilise RISET 105 kΩ** |

---

# 13. Fichiers de travail associés

```text
AdaptateurEcran(5).zip
commit interne observé : 95d58b6 feat(schematic): reintegrate isolated TPS61194 backlight

source primaire écran :
772288625-XT2175-x-Yukon-Moto-g200-5G-MB-Schematics-L3-Repair.pdf
page 86/90 : DISP: CONNECTOR / J1891

source Q6A :
radxa_dragon_q6a_schematic_v1.21.pdf
radxa_dragon_q6a_components_placement_map_v1.21.pdf
```

Un fichier séparé `MODIFICATION_SCHEMA_V1_FINALISATION_2026-09-23.md` décrit les modifications KiCad à appliquer avant passage au PCB.
