# CDC — Carte Power / MCU V1 du Maker Phone Q6A

**Date :** 2026-09-30  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** cahier des charges de référence pour le schéma de la carte Power/MCU V1  
**Priorité documentaire :** ce document supersède les choix contradictoires présents dans `PROJECT_STATE_POWER_USB_C_V1_2026-09-28.md`, `PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md` et les documents antérieurs concernant la carte Power/MCU.

---

# 1. Objectif de la carte

La carte V1 regroupe :

```text
- gestion batterie 1S
- charge USB-C 5 V
- vrai power-path système
- supervision always-on par MCU
- alimentation / coupure physique du Q6A
- deux sorties puissance auxiliaires commutées
- interface de contrôle du modem EC25
- interface USB 2.0 Q6A <-> EC25
- commandes d'activation écran, sans coupure physique DISPLAY_PWR dédiée
- connectique de debug / programmation / logique
```

La carte doit rester compatible avec **JLCPCB Economic PCBA**. La règle de sélection est :

```text
1. Basic
2. Promotional Extended
3. Extended seulement si nécessaire
```

Le nombre de références Extended différentes doit être minimisé.

---

# 2. Architecture globale figée

```text
                         USB-C 5 V
                            |
                            v
                  BQ25628E chargeur +
                   NVDC power-path
                      |         |
                     SYS       BAT
                      |         |
        +-------------+         +---- Li-ion/LiPo 1S
        |                               + NTC
        |
        +--> MAIN_PWR MOS -----------> Q6A / J19
        |
        +--> ANNEXE1_PWR MOS --------> sortie + / -
        |
        +--> ANNEXE2_PWR MOS --------> sortie + / -
        |
        +--> domaine always-on ------> TPS610995 -> RP2040-Tiny

Q6A USB2 host
  VBUS / D+ / D- / GND -------------> EC25 carrier

RP2040-Tiny
  <-> Q6A GPIO wake/sleep
  <-> BQ25628E I2C/IRQ
  <-> EC25 UART + DTR/RI + PWK/RST/STATUS
  -> MAIN_PWR / ANNEXE1_PWR / ANNEXE2_PWR
  -> commandes écran EN
```

---

# 3. Batterie

[FIGÉ]

```text
Type          : Li-ion / LiPo 1S
Capacité cible: ~5000 mAh
Tension       : ~3,0...4,2 V selon cellule
Protection    : pack protégé PCM de préférence
Température   : NTC externe physiquement en contact avec la cellule
```

La sécurité de charge ne doit jamais dépendre d'Android ou du firmware MCU.

Le chargeur, le NTC, les protections cellule et les limites matérielles doivent conserver un comportement sûr même si le MCU ou le Q6A est bloqué.

Un footprint de fusible série batterie peut être prévu avec possibilité de monter un strap 0 ohm pendant le bring-up. Le fusible protège surtout les défauts de surintensité / court-circuit ; il ne remplace pas la régulation CC/CV ni les protections de charge.

---

# 4. Chargeur / power-path

[FIGÉ]

Le chargeur retenu est :

```text
BQ25628ERYKR
JLCPCB : C18221178
famille : BQ25628E
assemblage : compatible Economic PCBA
charge max : 2 A
batterie : 1S
power-path : NVDC intégré
BATFET : intégré
I2C : oui
ADC / télémétrie : oui
```

Le choix de la variante `E` est volontaire : elle conserve les fonctions utiles au téléphone tout en étant confirmée compatible Economic PCBA. La fonction OTG boost du BQ25628 non-E n'est pas requise pour la V1.

Le BQ25628E remplace le BQ25892 précédemment envisagé.

## 4.1 Mesure batterie

[FIGÉ]

Le shunt externe + ampli de courant + ADC MCU est supprimé.

La V1 exploite directement la télémétrie du BQ25628E via I2C :

```text
VBAT
VSYS
VBUS
IBUS
IBAT
état de charge
NTC / température
faults
```

Limitation acceptée : la mesure IBAT n'est pas considérée comme un coulomb counter parfait pendant certaines phases de Battery Supplement lorsque VBUS est présent et que la batterie aide temporairement le système. Cette erreur est acceptée pour la V1 ; elle sera corrigée logiciellement par recalage SOC, tension batterie, charge complète et phases de repos.

## 4.2 Rendement / thermique

[FIGÉ AU NIVEAU ARCHITECTURE]

La cible de charge nominale est <= 2 A. Dans cette plage, le BQ25628/BQ25622 ont des rendements proches ; le BQ25628E est retenu pour sa limite naturelle à 2 A, sa simplification et sa disponibilité.

La validation thermique réelle reste obligatoire au bring-up avec :

```text
- 0,5 A
- 1 A
- 1,5 A
- 2 A
- Q6A actif simultanément
```

Mesurer température du BQ, de l'inductance, du connecteur USB-C et de la cellule.

---

# 5. BOM externe BQ25628E

[ARCHITECTURE FIGÉE — QUELQUES RÉFÉRENCES PASSIVES À CONFIRMER AVANT COMMANDE]

Valeurs de travail retenues d'après l'application TI :

```text
L buck      : 1 uH
C VBUS      : 1 uF
C PMID      : 10 uF
C PMID HF   : 100 nF
C SYS       : 2 x 10 uF
C BAT       : 10 uF
C REGN      : 4,7 uF
C BTST-SW   : 47 nF >= 10 V
pull-up I2C : 10 kOhm si nécessaires
pull-up INT : 10 kOhm
```

Références Basic déjà retenues / candidates :

```text
10 uF / 25 V X5R 0805 : C15850 Basic
1 uF                  : C52923 Basic
4,7 uF                : C1779 Basic
100 nF                : C1525 Basic
10 kOhm               : C25744 Basic
```

Inductance 1 uH haute intensité : référence à confirmer au gel BOM en privilégiant Economic et marge en saturation. Candidat de travail : `C22471110`, 1 uH, ~4 A nominal / 5,6 A saturation.

Le condensateur bootstrap 47 nF peut rester Extended si aucune référence Basic/Promo correcte n'est disponible.

Le réseau NTC doit être recalculé avec le modèle de NTC réellement choisi ; ne pas remplacer arbitrairement les valeurs du datasheet uniquement pour économiser une référence Extended si cela déplace les seuils thermiques.

---

# 6. USB-C extérieur

[FIGÉ : 5 V UNIQUEMENT]

La V1 ne négocie pas le Power Delivery.

```text
USB-C source -> 5 V uniquement
pas de CH224 / contrôleur PD
fallback universel 5 V = mode normal
```

Connecteur retenu :

```text
HRO TYPE-C-31-M-12
JLCPCB : C165948
USB-C femelle
16 pins / USB 2.0
Economic PCBA
```

Les lignes D+/D- du connecteur extérieur sont destinées au Q6A.

À prévoir autour du connecteur :

```text
- CC1 / CC2 en mode sink USB-C
- ESD sur D+ / D-
- protection VBUS adaptée
- routage USB2 différentiel propre
```

Les résistances CC, protections ESD et éventuel CMC restent à sélectionner en Basic/Promo si possible.

---

# 7. MCU superviseur

[FIGÉ POUR V1]

Le MCU est un **module RP2040-Tiny soudé directement sur la carte**, et non un RP2040 nu intégré au PCB.

Il reste le superviseur always-on du téléphone.

Responsabilités :

```text
- bouton Power
- demande suspend / wake Q6A
- séquences ON/OFF Q6A
- commandes MAIN_PWR / ANNEXE1 / ANNEXE2
- gestion BQ25628E par I2C
- gestion EC25
- supervision watchdog
- gestion future boutons / logique auxiliaire
```

Tout allumage / extinction demandé par l'utilisateur passe par le MCU. Le bouton Power n'agit pas directement sur le Q6A.

---

# 8. Alimentation always-on du RP2040-Tiny

[FIGÉ]

```text
BAT/SYS
  -> TPS610995DRVR
  -> 3,6 V
  -> VSYS RP2040-Tiny
  -> LDO 3,3 V embarqué
  -> RP2040 / GPIO 3,3 V
```

Le 3,3 V exact n'est pas injecté sur VSYS : on conserve une marge pour le LDO embarqué.

Composant :

```text
TPS610995DRVR
LCSC/JLC : C2071098
sortie fixe : 3,6 V
Iq très faible
```

BOM de travail :

```text
L : 2,2 uH
Cin : 10 uF
Cout : 2 x 10 uF
```

Les condensateurs doivent être Basic lorsque possible. L'inductance peut rester Extended si aucune Basic/Promo correcte n'offre le courant de saturation requis.

---

# 9. Q6A <-> MCU

[FIGÉ POUR LE PROTOTYPE]

Le header GPIO Q6A est en logique 3,3 V. Le RP2040-Tiny fonctionne lui aussi en logique 3,3 V.

Aucun level-shifter n'est requis entre ces GPIO.

```text
Q6A pin 34 : GND
Q6A pin 36 : GPIO59 = MCU_WAKE
Q6A pin 37 : GPIO58 = MCU_SLEEP_REQ
```

GPIO59 est la ligne de wake candidate à valider en vrai STR deep. GPIO58 sert à demander un suspend propre via le chemin logiciel Android/Linux.

Le vrai power-on depuis OFF complet est assuré par la logique de la carte Power/MCU et non par ces deux GPIO seuls.

---

# 10. MAIN_PWR Q6A

[FIGÉ ARCHITECTURE]

La branche Q6A doit pouvoir être mise hors tension physiquement par le MCU.

MOS P-channel retenu comme candidat principal :

```text
JMTQ55P02A
JLCPCB : C2890429
P-MOS
RDS(on) ~12 mOhm @ VGS = -2,5 V
PDFN 3,3 x 3,3 mm
Economic PCBA
```

Commande de gate :

```text
MCU -> résistance -> S8050 NPN -> gate P-MOS
pull-up gate -> source
```

Transistor de commande :

```text
S8050 / C2146
Basic
```

Le rail doit rester OFF par défaut pendant reset / absence de commande du MCU.

Le MOS ne doit pas servir à l'arrêt normal brutal : Android doit être arrêté proprement avant coupure physique, sauf récupération après blocage.

---

# 11. Annexes puissance

[FIGÉ]

Deux branches auxiliaires indépendantes sont prévues :

```text
ANNEXE1_PWR
ANNEXE2_PWR
```

Chaque canal n'expose que :

```text
+
-
```

Les signaux I2C/GPIO/logique seront disponibles sur un connecteur logique séparé et ne sont pas dupliqués sur les connecteurs puissance.

Pour simplifier la BOM, le même P-MOS `JMTQ55P02A / C2890429` peut être utilisé pour MAIN_PWR, ANNEXE1 et ANNEXE2. Cela n'ajoute aucune nouvelle référence Extended et donne une forte marge de courant.

Commande de chaque rail par S8050 Basic, même topologie que MAIN_PWR.

Ces annexes sont destinées notamment à :

```text
- audio
- haptique
- capteurs / modules externes
- futurs sous-ensembles
```

---

# 12. Écran

[FIGÉ]

Aucun `DISPLAY_PWR` high-side dédié n'est ajouté en V1.

La gestion de consommation écran utilise les entrées d'activation existantes :

```text
BL_EN
ENP
ENN
séquences panel / DSI
```

Cette décision réduit la BOM et le routage. Une coupure physique globale pourra être réintroduite ultérieurement si les mesures montrent une consommation résiduelle problématique.

---

# 13. EC25 — liaison USB Q6A

[FIGÉ]

Le connecteur micro-USB physique du carrier EC25 ne sera pas utilisé dans le téléphone.

Les signaux USB exposés par le carrier sont câblés directement au Q6A :

```text
Q6A USB2 host      EC25 carrier
--------------------------------
VBUS 5 V --------> VBUS
D+ --------------> DP
D- --------------> DN
GND --------------> GND
```

Le carrier EC25 a été observé fonctionnel alimenté uniquement par cette liaison USB du Q6A. Pour la V1 :

```text
MODEM_PWR séparé : supprimé
VIN/BAT carrier   : non utilisés en fonctionnement normal
```

Lorsque MAIN_PWR coupe physiquement le Q6A, le VBUS USB Q6A disparaît également et le modem est donc mis hors tension.

La tenue du VBUS Q6A lors des pics LTE réels devra être validée avec SIM, data et appels.

---

# 14. EC25 — interface MCU

[FIGÉ ARCHITECTURE]

Le carrier expose les signaux fonctionnels du modem. Le EC25 nu travaille principalement en logique 1,8 V ; le RP2040 est en 3,3 V.

Les signaux retenus MCU <-> EC25 sont :

```text
TXD
RXD
DTR
RI
PWK
RST
STATUS
VIO
GND
```

## 14.1 TXD / RXD / DTR / RI

Level-shifter retenu :

```text
SN74AVC4T245DR
JLCPCB : C22495
4 canaux
2 groupes de 2 bits avec directions indépendantes
VCCA = VIO EC25 ~1,8 V
VCCB = 3V3 RP2040
```

Répartition :

```text
EC25 -> MCU : TXD, RI
MCU -> EC25 : RXD, DTR
```

Découplage : 100 nF Basic sur chaque rail d'alimentation du translateur.

La tension `VIO` réelle du carrier doit être mesurée avant branchement définitif. Attendu : ~1,8 V.

## 14.2 PWK / RST

Pas de level-shifter push-pull.

Commandes en open-collector via NPN Basic :

```text
MCU GPIO -> résistance -> S8050 -> PWK
MCU GPIO -> résistance -> S8050 -> RST
```

`PWK` sert aux séquences normales ON/OFF modem.

`RST` est réservé à la récupération en cas de modem bloqué.

## 14.3 STATUS

`STATUS` est ajouté au câblage MCU.

Il est traité comme sortie open-drain du modem avec pull-up vers le 3,3 V MCU :

```text
STATUS EC25 -> GPIO MCU
pull-up ~10 kOhm -> 3V3 MCU
```

Il permet au MCU de vérifier matériellement l'état du modem et d'éviter les séquences basées uniquement sur des temporisations fixes.

## 14.4 Signaux non requis vers le MCU

Ne sont pas nécessaires au superviseur V1 :

```text
RTS / CTS
DCD
NET_STATUS / NET_MODE
W_DISABLE#
AP_READY / WAKEUP_IN
```

`USB_BOOT` peut être prévu en testpoint uniquement.

Les lignes PCM sont réservées à l'audio futur. SDA/SCL/AD0/AD1 peuvent être exposés sur la connectique logique générale si utile, mais ne font pas partie du lien MCU-modem minimal.

---

# 15. Connectique logique séparée

[FIGÉ PRINCIPE]

Une connectique distincte des sorties puissance doit permettre d'accéder aux signaux logiques nécessaires aux extensions et au debug.

À inclure selon place disponible :

```text
GND
3V3 MCU
I2C SDA
I2C SCL
GPIO libres
UART debug MCU
reset / programmation
```

Les connecteurs ANNEXE1/2 restent strictement puissance `+/-`.

---

# 16. Debug / bring-up obligatoire

La PCB doit fournir des points de test ou pads accessibles pour au minimum :

```text
BAT
SYS
VBUS USB-C
3V6 MCU
3V3 MCU
GND
BQ INT / I2C
MAIN_PWR gate / sortie
ANNEXE1 sortie
ANNEXE2 sortie
EC25 VIO
EC25 TXD
EC25 RI
EC25 STATUS
Q6A GPIO58 / GPIO59
```

Prévoir des possibilités de bypass des étages de puissance pendant le bring-up lorsque cela ne crée pas de risque de contention.

---

# 17. Références principales gelées

```text
Chargeur / power-path : BQ25628ERYKR      C18221178  Extended / Economic
MCU module            : RP2040-Tiny       module soudé au PCB
Alim MCU              : TPS610995DRVR     C2071098   Extended
Level-shift EC25      : SN74AVC4T245DR    C22495     Extended
P-MOS puissance       : JMTQ55P02A        C2890429  Extended / Economic
NPN commande          : S8050             C2146      Basic
USB-C                 : TYPE-C-31-M-12    C165948    Extended / Economic
10 uF / 25 V          : C15850                       Basic
1 uF                  : C52923                       Basic
4,7 uF                : C1779                        Basic
100 nF                : C1525                        Basic
10 kOhm               : C25744                       Basic
```

Les statuts de stock et de classification JLCPCB doivent être revalidés juste avant génération de la BOM de commande.

---

# 18. Décisions anciennes explicitement supplantées

```text
BQ25892 candidat principal
    -> SUPPLANTÉ par BQ25628E

shunt batterie + ampli courant externe
    -> SUPPRIMÉ ; télémétrie BQ25628E utilisée

USB-C PD 9 V envisagé
    -> SUPPRIMÉ V1 ; 5 V uniquement

MODEM_PWR séparé
    -> SUPPRIMÉ ; carrier alimenté par VBUS USB Q6A

micro-USB physique carrier EC25
    -> NON utilisé ; VBUS/DP/DN/GND câblés directement

DISPLAY_PWR physique
    -> SUPPRIMÉ V1 ; commandes EN uniquement

MCU exact non choisi
    -> RP2040-Tiny soudé comme module sur la PCB

EC25 supposé directement compatible 3,3 V
    -> NON retenu comme hypothèse générale ; TXD/RXD/DTR/RI passent par SN74AVC4T245
```

---

# 19. Points restant à figer avant schéma final

Il ne reste plus de choix d'architecture majeur. Les points restants sont principalement des références et dimensionnements :

```text
[ ] ILIM et stratégie de courant d'entrée BQ25628E sur USB-C 5 V
[ ] NTC exact et réseau de seuils
[ ] inductance BQ25628E finale
[ ] condensateur bootstrap 47 nF final
[ ] résistances CC1/CC2 USB-C
[ ] ESD USB D+/D-/CC/VBUS
[ ] fusible BAT ou strap 0 ohm
[ ] valeurs exactes résistances de base/gate des drivers MOS
[ ] connecteurs batterie / Q6A / annexes / logique
[ ] footprint mécanique précis du module RP2040-Tiny
[ ] mesure réelle EC25 VIO/TXD/RI pour valider 1,8 V carrier
[ ] dimensions PCB et 2 couches vs 4 couches
```

---

# 20. Critères de validation V1

La carte V1 sera considérée fonctionnellement validée quand :

```text
[ ] USB-C 5 V alimente SYS sans batterie
[ ] batterie seule alimente le système
[ ] plug/unplug USB sans reboot système
[ ] charge 0,5 / 1 / 1,5 / 2 A validée thermiquement
[ ] NTC arrête / limite correctement la charge hors plage
[ ] télémétrie BQ cohérente en I2C
[ ] RP2040-Tiny reste always-on de manière stable
[ ] MAIN_PWR démarre et coupe le Q6A de manière déterministe
[ ] ANNEXE1/2 commutent correctement
[ ] Q6A GPIO58/59 suspend/wake validés
[ ] EC25 USB fonctionne via câblage direct VBUS/DP/DN/GND
[ ] UART EC25 fonctionne à travers SN74AVC4T245
[ ] PWK/RST/DTR/RI/STATUS validés
[ ] aucun back-power problématique entre domaines
[ ] consommation deep-off mesurée
[ ] aucune référence n'oblige à quitter Economic PCBA
```
