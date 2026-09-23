# Modification schéma V1 — finalisation connecteur écran et gel prototype

**Date :** 2026-09-23  
**Projet concerné :** `AdaptateurEcran(5).zip`  
**But :** appliquer les dernières corrections avant de considérer le schéma V1 gelé et de passer au PCB.

---

# 1. Résumé des modifications obligatoires

Deux modifications fonctionnelles/documentaires restent à faire :

```text
1. corriger J2 : pin 1 = NC, pin 2 = DISP_IOVCC_1P8
2. ajouter un repère d’orientation physique côté pads électriques 39/40 = GND/GND
```

Le backlight TPS61194 déjà réintégré dans `AdaptateurEcran(5)` est conservé comme architecture principale. Il ne faut pas revenir au SY7203 Q6A comme chemin normal.

---

# 2. Source de vérité pour le pinout écran

Utiliser le schéma Motorola Moto G200/Yukon :

```text
772288625-XT2175-x-Yukon-Moto-g200-5G-MB-Schematics-L3-Repair.pdf
page 86 / 90
feuille : DISP: CONNECTOR
connecteur : J1891
```

Le marquage physique `1 / 20 / 21 / 40` imprimé sur le flex aftermarket ne doit pas être pris comme numérotation électrique du connecteur. Il sert seulement de repère mécanique.

Mesure réelle obtenue sur l’écran :

```text
extrémité marquée 40 / 1  : continuité entre les deux contacts d’extrémité
extrémité marquée 21 / 20 : aucune continuité entre les deux contacts d’extrémité
```

Cette observation est cohérente avec la numérotation électrique retenue :

```text
extrémité électrique 39 / 40 = GND / GND
extrémité électrique 1 / 2   = NC / IOVCC 1,8 V
```

---

# 3. Correction du symbole écran J2

## 3.1 Fichier librairie

Modifier :

```text
V1/interposer_q6a.kicad_sym
```

Symbole concerné :

```text
Moto_G200_Display_40P
```

ou le nom exact équivalent actuellement utilisé par J2.

### État actuel incorrect

```text
pin 1 : DISP_IOVCC_1P8
pin 2 : NC
```

### État cible

```text
pin 1 : NC
pin 2 : DISP_IOVCC_1P8
```

Types électriques recommandés :

```text
pin 1 : no_connect / passive selon convention du symbole, puis marqueur NC dans le schéma
pin 2 : passive
```

Ne modifier aucun autre numéro de pin lors de cette opération.

---

# 4. Correction de la feuille 01_Interface_Display

Modifier :

```text
V1/01_Interface_Display.kicad_sch
```

J2 est le connecteur écran :

```text
Reference : J2
Footprint : interposer_q6a:Panasonic_AXE540127_40P_P0.4mm
```

Après mise à jour du symbole depuis la librairie :

```text
J2.1 -> NC
J2.2 -> DISP_IOVCC_1P8
```

Actions précises :

1. supprimer toute connexion/net `DISP_IOVCC_1P8` actuellement attachée à J2.1 ;
2. placer un marqueur No Connect sur J2.1 ;
3. supprimer le marqueur No Connect actuellement placé sur J2.2 ;
4. raccorder J2.2 au net `DISP_IOVCC_1P8` ;
5. conserver le même net 1,8 V déjà utilisé par le reste du design ;
6. vérifier qu’aucune étiquette locale/globale orpheline ne reste sur la pin 1.

Ne pas renuméroter le footprint pour corriger cette erreur : la correction se fait dans le **mapping fonctionnel du symbole**, pas en inversant artificiellement les pads physiques.

---

# 5. Pinout J2 à contrôler après correction

Le contrôle netlist doit retrouver exactement :

```text
 1  NC
 2  DISP_IOVCC_1P8
 3  DISP_VSP
 4  DISP_RST_N
 5  DISP_VSN
 6  DISP_TE
 7  NC
 8  DISP_PWM_OUT
 9  NC
10  GND
11  PANEL_WLED_A / DISP_WLED_A
12  DSI_D2_P
13  PANEL_WLED_K1 / DISP_WLED_K1
14  DSI_D2_N
15  PANEL_WLED_K2 / DISP_WLED_K2
16  GND
17  PANEL_WLED_K3 / DISP_WLED_K3
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

Les masses fonctionnelles doivent être :

```text
10, 16, 22, 28, 29, 34, 39, 40
```

---

# 6. Footprint AXE540127 — conserver la numérotation existante

Fichier :

```text
V1/interposer_q6a.pretty/Panasonic_AXE540127_40P_P0.4mm.kicad_mod
```

La géométrie actuelle donne :

```text
pads impairs : y = -1,2 mm
pads pairs   : y = +1,2 mm

extrémité X négative : pads 1 / 2
extrémité X positive : pads 39 / 40
```

Exemples actuels :

```text
pad 1  : x = -3,8 ; y = -1,2
pad 2  : x = -3,8 ; y = +1,2
pad 39 : x = +3,8 ; y = -1,2
pad 40 : x = +3,8 ; y = +1,2
```

**Ne pas inverser ni renuméroter ces pads.**

Le footprint est conservé en tant que socket femelle Panasonic AXE540127.

---

# 7. Ajouter un repère d’orientation dédié côté masses

## 7.1 Pourquoi

Le flex aftermarket imprime `1/20/21/40`, mais cette inscription n’est pas suffisamment fiable pour orienter électriquement l’écran.

On utilise donc un repère PCB propre au projet, basé sur une propriété électrique robuste :

```text
pads électriques 39 et 40 = GND + GND
```

## 7.2 Repère à ajouter

Sur le footprint J2, ajouter côté pads `39 / 40`, donc côté **X positif**, un point plein visible :

```text
●
```

Dimension recommandée :

```text
Ø ~1,0 mm
```

Le point doit être sur `F.SilkS` et ne doit pas recouvrir :

- les pads 39/40 ;
- les pads mécaniques 42/44 ;
- le courtyard ;
- la zone physique nécessaire à l’accouplement.

Position exacte à ajuster dans l’éditeur selon les règles de sérigraphie JLCPCB.

Ajouter si possible à côté :

```text
39/40 GND
```

ou, si l’espace est trop faible :

```text
GND END
```

Répéter le repère sur `F.Fab`.

## 7.3 Ne pas supprimer le marqueur pin 1

Le footprint actuel possède déjà un petit cercle standard du côté pin 1 :

```text
centre approximatif : X = -5,85 ; Y = -1,20
```

Ce marqueur doit rester.

Le design comportera donc deux informations distinctes :

```text
petit marqueur standard côté 1/2 : repère pin 1 du composant
point utilisateur ● côté 39/40 : sens d’insertion de l’écran / extrémité GND
```

Pour éviter toute confusion, le point côté 39/40 doit être accompagné du texte `GND`, `39/40` ou `GND END`.

---

# 8. Ajouter une note visible dans la feuille d’interface

Dans `01_Interface_Display.kicad_sch`, ajouter une note de bring-up proche de J2 :

```text
J2 ORIENTATION:
- electrical end 39/40 = GND/GND -> PCB mark ● GND END
- electrical end 1/2 = NC/IOVCC_1P8
- flex marks 1/20/21/40 are NOT used as electrical numbering
- no hot-plug
```

But : empêcher une future réinterprétation du sens à partir du marquage du flex.

---

# 9. Backlight — ne pas revenir à l’ancienne architecture

La feuille :

```text
V1/04_Backlight.kicad_sch
```

contient déjà le TPS61194 réintégré dans `AdaptateurEcran(5)`.

Le conserver comme chemin principal.

## 9.1 État cible TPS61194

```text
U2 = TPS61194PWPR
L2 = 4,7 µH
D1 = Schottky 60 V / 3 A
Cin = 3 × 10 µF / 25 V
Cout = 3 × 10 µF / 100 V
RFSET = 27,4 kΩ
RISET = 105 kΩ
Istring cible ≈ 24,8 mA
VBOOST_MAX ≈ 30,9 V
OUT1 -> K1
OUT2 -> K2
OUT3 -> K3
OUT4 -> GND selon configuration retenue
```

Ne pas réintroduire l’ancien `RISET 94,2 kΩ / 95,3 kΩ`.

## 9.2 Sélection BL_VIN

Conserver :

```text
Q6A_5V_IN -- R10 0R DNI --+
                         +-- BL_VIN
BL_VIN_EXT -- R11 0R DNI --+
```

Les deux résistances restent DNI par défaut.

Une seule peut être montée à la fois.

`Q6A_5V_IN` reste un fil externe vers `VCC_5V_PERI` du Q6A et ne doit pas être représenté comme provenant de J10.

## 9.3 Backlight Q6A natif

Conserver le chemin natif uniquement comme fallback expérimental :

```text
J10 LED+/LED-
-> straps DNI
-> ballasts 47 Ω DNI
-> panneau
```

Il doit rester isolé en mode TPS61194.

R315 et C644 ne doivent pas être modifiés pour le mode normal TPS61194.

C644 réel a été mesuré à environ 2,2 µF in-circuit ; sa tension nominale reste inconnue. Cette incertitude n’est pas bloquante pour le chemin principal TPS61194.

---

# 10. Bias LCD — conserver l’état actuel

Aucune modification fonctionnelle supplémentaire demandée dans cette mission.

Conserver :

```text
TPS65132WRVCT
VIN = LCD_3V3
ENP / ENN séparés
100 kΩ pull-down sur ENP/ENN
TP filaires LCD_3V3 / ENP / ENN
TP mesure VSP / VSN
J3 GND/SDA/SCL/VIN_3V3
VSP cible +5,5 V
VSN cible -5,7 V
```

---

# 11. Tactile — conserver l’état actuel

Le tactile principal reste :

```text
Novatek SPI 1,8 V
4 × 74LVC2T45
Q6A SPI6 + GPIO RESET/IRQ
```

Le breakout J4 du tactile natif Q6A reste passif et séparé.

Aucune modification tactile n’est demandée par ce document.

---

# 12. BOM

La correction J2.1/J2.2 ne change pas la BOM.

Le repère sérigraphie ne change pas la BOM.

J2 reste :

```text
Panasonic AXE540127
LCSC C3652918
```

Le backlight reste celui de `AdaptateurEcran(5)` avec TPS61194 et les composants déjà choisis.

Avant commande, recontrôler disponibilité LCSC/JLC et compatibilité Economic PCBA pour les composants extended.

---

# 13. Vérifications KiCad obligatoires après modification

## 13.1 ERC

Relancer :

```text
ERC complet du projet
```

Critère :

```text
0 erreur
0 avertissement non justifié
```

Ne pas réutiliser comme preuve l’ERC précédent à la correction J2.1/J2.2.

## 13.2 Netlist J2

Contrôler explicitement :

```text
J2.1  = NC
J2.2  = DISP_IOVCC_1P8
J2.10 = GND
J2.16 = GND
J2.22 = GND
J2.28 = GND
J2.29 = GND
J2.34 = GND
J2.39 = GND
J2.40 = GND
```

Puis :

```text
J2.11 = WLED_A
J2.13 = K1
J2.15 = K2
J2.17 = K3
```

Et les cinq paires DSI :

```text
12/14 = D2 +/-
18/20 = D1 +/-
24/26 = CLK +/-
30/32 = D0 +/-
36/38 = D3 +/-
```

## 13.3 Footprint

Dans PCB Editor / Footprint Editor :

```text
[ ] pad 1 côté X négatif
[ ] pad 2 côté X négatif
[ ] pad 39 côté X positif
[ ] pad 40 côté X positif
[ ] pin-1 marker standard toujours visible
[ ] nouveau ● GND END visible côté 39/40
[ ] aucun overlap masque/paste/pad
[ ] courtyard non violé par la géométrie physique
```

---

# 14. Contrôle avant fabrication

Checklist finale :

```text
[ ] pin 1 = NC
[ ] pin 2 = IOVCC 1,8 V
[ ] pinout Motorola complet inchangé ailleurs
[ ] masses = 10/16/22/28/29/34/39/40
[ ] AXE540127 femelle conservé
[ ] repère ● côté 39/40 ajouté
[ ] note d’orientation ajoutée au schéma
[ ] TPS61194 = voie normale
[ ] BL_VIN Q6A/externe mutuellement exclusifs et DNI par défaut
[ ] SY7203 natif isolé et DNI par défaut
[ ] bias inchangé
[ ] tactile inchangé
[ ] ERC propre
[ ] netlist contrôlée
[ ] BOM contrôlée
[ ] Journal/JOURNAL.md mis à jour
```

---

# 15. Entrée à ajouter au journal du projet AdaptateurEcran

Ajouter dans `Journal/JOURNAL.md` :

```text
## 2026-09-23 — Correction pinout J2 et repère d’orientation écran

- Source primaire Motorola page 86 `DISP: CONNECTOR / J1891` reprise pour lever l’erreur historique J2.1/J2.2 : pin 1 = NC, pin 2 = DISP_IOVCC_1P8.
- Le marquage 1/20/21/40 du flex aftermarket n’est plus utilisé comme numérotation électrique.
- Inspection réelle : écran FMS780 JB2-1, connecteur mâle 40 contacts. Continuité constatée entre les deux contacts de l’extrémité du flex marquée 40/1, aucune continuité entre ceux de l’extrémité 21/20 ; cohérent avec une extrémité électrique 39/40 = GND/GND.
- Ajout sur le footprint AXE540127 d’un repère utilisateur `● 39/40 GND` côté masses, en conservant le marqueur standard pin 1 côté opposé.
- TPS61194 reste le chemin backlight principal ; SY7203 Q6A reste un fallback isolé par DNI. R315/C644 ne sont pas modifiés pour le mode normal.
- ERC et netlist à relancer après correction avant gel PCB.
```

---

# 16. Critère de fin de mission

La mission est terminée lorsque :

```text
1. J2.1/J2.2 corrigés dans la librairie et dans la feuille 01
2. footprint J2 porte le repère ● 39/40 GND
3. note d’orientation présente dans le schéma
4. ERC propre
5. netlist validée
6. journal mis à jour
7. aucune régression sur bias, tactile ou TPS61194
```

À ce moment, le **schéma V1 est gelé pour prototype** et le travail peut passer au placement/routage PCB.
