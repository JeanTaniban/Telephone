# PROJECT STATE — PCB Power / Batterie / USB-C V1

**Date :** 2026-09-28  
**Statut :** document de conception — décisions figées + options encore ouvertes  
**Projet :** Maker Phone sur Radxa Dragon Q6A V1.21  
**Document power précédent :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-25_NATIVE_BATTERY_USB_C.md`

> Ce document décrit la **carte externe Power / Batterie / USB-C V1** à concevoir et commander chez JLCPCB en même temps que la carte d'adaptation écran. Il complète le document du 25/09. Il ne remplace pas les faits déjà validés concernant le rework Q6A/J19 ; il précise l'architecture de la carte externe et distingue explicitement ce qui est figé de ce qui reste à sélectionner/valider.

---

# 1. Objectif de la carte V1

La carte doit devenir le **power manager / power supervisor** du téléphone.

Objectifs :

```text
USB-C extérieur
  |
  +--> USB 2.0 D+/D- --------------------------> Q6A
  |
  +--> gestion Type-C / éventuellement PD
  |
  +--> chargeur 1S + vrai power-path
            |
            +--> batterie Li-ion/LiPo 1S
            |
            +--> SYS / rail système
                    |
                    +--> Q6A
                    +--> modem EC25
                    +--> autres rails commandés

Domaine always-on :
MCU + mesure batterie + supervision alimentation
```

La V1 doit rester **simple à fabriquer, simple à mettre au point et sûre**, mais elle doit préparer l'architecture finale du téléphone et éviter une V1 jetable qui obligerait à refaire immédiatement toute la gestion d'alimentation.

Contraintes de coût/fabrication :

- assemblage JLCPCB ;
- utiliser autant que possible des composants **Basic** ou **Promotional Extended** ;
- accepter quelques composants Extended lorsque leur fonction est réellement critique ;
- éviter les boîtiers complexes à assembler en V1 si une alternative QFN/DFN raisonnable existe ;
- prévoir des straps/jumpers/testpoints pour pouvoir isoler ou bypasser les sous-ensembles lors du bring-up.

---

# 2. Décisions FIGÉES

## 2.1 Batterie 1S et alimentation du Q6A par J19

[FIGÉ]

Le téléphone utilise une batterie **Li-ion/LiPo 1S**, environ :

```text
3,0...4,2 V utile selon cellule
3,7 / 3,8 V nominal
```

Le Q6A est alimenté par son entrée batterie native **J19 / VBATT_PWR**.

Le Q6A ne doit pas être alimenté dans le téléphone final par un chemin du type :

```text
1S -> boost 5/9/12 V -> J21 -> reconversion interne
```

On conserve le chemin batterie natif et efficace du Q6A.

## 2.2 Charge interne du Q6A non utilisée

[FIGÉ]

La fonction charge du PM7250B n'est **pas réactivée**. Le rework serait inutilement lourd et la gestion batterie sera maîtrisée par notre carte externe.

Configuration Q6A cible déjà documentée :

```text
R7    -> DNP / retirée
R24   -> 2 mΩ 1 %
R190  -> 100 kΩ vers GND
R191  -> 10 kΩ vers GND
FB4   -> DNP
R185..R189 -> DNP si inspection physique confirme
```

Rôles :

```text
R7    : suppression du bypass batteryless
R24   : fermeture J19 -> VBATT_PWR + shunt conforme au schéma Q6A
R190  : température batterie simulée valide côté PM7250B
R191  : batterie présente + charge PM7250B désactivée
FB4   : charge interne physiquement non activée
```

Le footprint exact de R24 doit toujours être mesuré avant achat de la résistance finale 2 mΩ.

## 2.3 USB 2.0 suffisant en V1

[FIGÉ]

La carte externe doit exposer et router :

```text
VBUS
GND
CC1
CC2
D+
D-
```

Les lignes SuperSpeed ne sont pas nécessaires en V1.

D+/D- servent au Q6A pour :

- ADB ;
- EDL / debug selon configuration ;
- MTP / USB device ;
- développement Android.

SBU1/SBU2 et les paires SuperSpeed ne sont pas requises pour la première version.

## 2.4 Compatibilité chargeur non-PD obligatoire

[FIGÉ]

Le téléphone doit charger sur une source USB-C **5 V classique**, même en absence de Power Delivery.

Séquence attendue :

```text
branchement USB
      |
      +--> 5 V disponible
      |
      +--> si source PD compatible : négociation éventuelle d'une tension supérieure
      |
      +--> sinon : rester à 5 V et charger normalement à puissance réduite
```

Le PD ne doit jamais être nécessaire au fonctionnement de base.

## 2.5 Vrai power-path souhaité pour la carte intégrée

[FIGÉ ARCHITECTURE]

La carte V1 intégrée doit viser un **vrai power-path / SYS**, et non simplement un chargeur connecté sur le même nœud que la batterie et le système.

Architecture recherchée :

```text
USB
 |
chargeur + power-path
 |
 +--> SYS ----> Q6A / système
 |
 +--> BAT ----> batterie
```

Comportement voulu :

- USB présent : le système est alimenté principalement depuis l'USB ;
- le surplus charge la batterie ;
- si le système demande momentanément plus que l'entrée ne peut fournir, la batterie peut compléter ;
- USB retiré : la batterie reprend l'alimentation sans reboot ;
- le courant système n'est pas confondu avec le courant de charge batterie.

Un IP2312 simple reste acceptable comme **module de prototype séparé**, mais n'est plus la cible principale pour notre propre PCB.

## 2.6 Sécurité indépendante du logiciel

[FIGÉ]

La sécurité de charge ne doit **jamais dépendre d'Android ou du MCU**.

Même si le Q6A est planté, éteint ou suspendu, le chargeur doit rester sûr matériellement.

La chaîne doit inclure au minimum :

- régulation CC/CV autonome ;
- limite de courant d'entrée ;
- limite de courant de charge ;
- surveillance température batterie via NTC ;
- protections surtension / sous-tension / thermique fournies par les IC retenus ;
- batterie avec PCM/protection cellule si possible ;
- fusible ou protection matérielle à définir dans le chemin batterie ;
- aucune dépendance au firmware pour éviter une surcharge cellule.

La protection intégrée à une cellule/pack protégé est un **dernier niveau de sécurité**, pas le chargeur principal.

## 2.7 MCU always-on comme superviseur d'énergie

[FIGÉ ARCHITECTURE]

Le MCU reste dans un domaine **always-on** à très faible consommation et devient le superviseur du téléphone.

Il doit pouvoir gérer :

```text
- chargeur via I²C
- télémétrie batterie
- coulomb counting
- température batterie
- bouton power / événements wake
- état USB / charge
- commande alimentation Q6A
- commande alimentation modem
- commande alimentation écran / sous-ensembles
- watchdog / récupération d'un sous-système bloqué
- transmission des informations batterie vers le Q6A / Android
```

Le MCU exact n'est pas encore choisi dans ce document.

## 2.8 Fuel gauge principal géré par MCU

[FIGÉ ARCHITECTURE]

Le fuel gauge Android ne doit pas dépendre uniquement du PM7250B du Q6A.

La stratégie retenue est :

```text
shunt batterie
   |
ampli de courant bidirectionnel
   |
ADC MCU
   |
intégration courant dans le temps
   |
SOC / mAh / diagnostic batterie
   |
Q6A / Android
```

Le chargeur fournit en parallèle ses informations par I²C : tension, état charge, défauts, températures et mesures disponibles selon la puce choisie.

Le MCU fusionne ces informations avec sa propre mesure batterie.

---

# 3. Power gating des sous-systèmes

## 3.1 Principe retenu

[FIGÉ ARCHITECTURE]

La carte doit prévoir la possibilité de **couper physiquement** les principaux sous-systèmes.

```text
SYS / BAT
   |
   +--> ALWAYS_ON ----> MCU + mesure batterie
   |
   +--> MAIN_PWR -----> Q6A
   |
   +--> MODEM_PWR ----> EC25
   |
   +--> DISPLAY_PWR --> écran / étage écran si nécessaire
   |
   +--> extensions futures : audio / haptique / capteurs
```

Les switches exacts restent à choisir.

## 3.2 Pourquoi couper physiquement

Objectifs :

- deep-off réel ;
- consommation de veille minimale ;
- récupération après blocage logiciel ;
- éviter le back-power par USB/GPIO ;
- séquencement propre des sous-systèmes ;
- possibilité d'isoler une panne ;
- arrêt matériel de sécurité.

## 3.3 Q6A

Le switch d'alimentation Q6A ne doit **pas** être utilisé comme méthode normale d'arrêt brutal.

Séquence normale :

```text
commande shutdown Android
        |
Q6A termine proprement
        |
MCU constate / attend l'arrêt
        |
MAIN_PWR OFF
```

Séquence de secours :

```text
Q6A bloqué
   |
watchdog / timeout MCU
   |
MAIN_PWR forcé OFF
```

Cette coupure de secours ne doit pas remplacer un shutdown Android propre, notamment pour éviter les écritures eMMC interrompues.

## 3.4 Modem EC25

Le modem doit idéalement avoir deux niveaux de contrôle :

```text
MCU -> PWRKEY / PEN / signaux modem
MCU -> MODEM_PWR hard switch
```

L'arrêt normal doit utiliser le mécanisme modem prévu.

Le hard switch sert à :

- deep-off ;
- récupération modem bloqué ;
- sécurité.

Il faudra aussi traiter le **VBUS USB du modem** afin d'éviter un back-power lorsque son rail principal est coupé.

## 3.5 Écran

L'arrêt normal de l'écran doit utiliser ses entrées d'activation existantes :

```text
BL_EN
ENP / ENN du bias
séquences panel
```

Un `DISPLAY_PWR` physique peut être prévu pour :

- deep-off ;
- récupération ;
- suppression de toute consommation résiduelle.

## 3.6 MOSFET vs load-switch

[À SÉLECTIONNER]

Un MOSFET high-side peut offrir des pertes très faibles :

```text
P = I² x RDS(on)
```

Exemple pour 10 mΩ :

```text
1 A -> 10 mW
2 A -> 40 mW
3 A -> 90 mW
5 A -> 250 mW
```

Cependant, pour certains rails, un vrai **load-switch / eFuse** peut être préférable car il peut apporter :

- soft-start ;
- limitation du courant d'appel ;
- reverse-current blocking ;
- protection thermique ;
- signal FAULT.

Pour les chemins susceptibles de subir un courant inverse ou du back-power, privilégier :

- switch avec reverse blocking ; ou
- deux MOSFETs dos-à-dos.

Prévoir des **jumpers de bypass** sur la V1 pour faciliter le bring-up.

---

# 4. Chargeur principal — candidats

Le composant principal n'est **pas encore figé**.

## 4.1 BQ25892 — candidat actuellement en tête

[À VALIDER / CANDIDAT PRINCIPAL]

Raisons :

- charge 1S jusqu'à environ 5 A ;
- vrai NVDC power-path ;
- Battery Supplement ;
- bonne efficacité aux courants élevés ;
- entrée compatible avec une alimentation 9 V ;
- I²C ;
- ADC / télémétrie chargeur ;
- NTC ;
- OTG disponible ;
- boîtier QFN raisonnable pour JLCPCB ;
- les lignes USB D+/D- peuvent rester dédiées au Q6A sur la variante BQ25892 ;
- disponible chez JLCPCB/LCSC lors de la recherche sous la référence **C165480**.

Limite identifiée :

- l'ADC interne n'est pas aussi intéressant qu'un BQ25622/BQ25638 pour mesurer précisément un courant batterie bidirectionnel en décharge ;
- cette limite devient peu importante puisque le MCU aura son propre shunt + ampli de courant.

La disponibilité, le prix et la classification JLCPCB doivent être revalidés juste avant commande.

## 4.2 BQ25622 — très bon candidat secondaire

[À VALIDER]

Avantages :

- 1S ;
- vrai power-path NVDC ;
- environ 3,5 A de charge max ;
- entrée jusqu'à environ 18 V ;
- fonctionnement autonome ;
- I²C ;
- ADC riche ;
- mesure `IBAT` bidirectionnelle intéressante pour le MCU ;
- NTC / protections / OTG.

Inconvénients par rapport au BQ25892 pour cette V1 :

- courant de charge max inférieur ;
- rendement à fort courant potentiellement un peu moins intéressant selon point de fonctionnement ;
- sourcing JLCPCB moins évident lors des recherches ;
- intérêt de son `IBAT` signé moins déterminant maintenant qu'un shunt externe est prévu.

## 4.3 BQ25638 — techniquement excellent, mais non retenu pour la V1

[NON RETENU V1]

Très intéressant techniquement :

- charge jusqu'à environ 5 A ;
- entrée haute tension ;
- power-path ;
- très bon rendement ;
- ADC riche ;
- mesure courant batterie bidirectionnelle ;
- faible résistance du BATFET.

Mais :

- boîtier DSBGA ~2 x 2,5 mm ;
- assemblage JLCPCB Standard / inspection plus contraignante ;
- complexité injustifiée pour une première version économique.

Peut être reconsidéré plus tard.

## 4.4 IP2312

[FALLBACK / PROTOTYPE]

Avantages :

- courant de charge élevé pour un module simple ;
- 5 V ;
- très répandu sur AliExpress ;
- simple pour valider rapidement une batterie 1S.

Limite importante :

- pas de vrai SYS/power-path tel que recherché pour la carte intégrée.

Il reste utile pour essais externes, mais n'est plus le choix préféré pour la PCB V1 si un BQ avec power-path reste accessible.

## 4.5 CN3791

[ÉCARTÉ POUR LA PCB V1]

Le composant accepte une large plage d'entrée et existe sur des modules chinois, mais les modules courants sont surtout orientés charge solaire/MPPT et l'architecture n'apporte pas le power-path recherché.

---

# 5. USB-C / Power Delivery

## 5.1 PD

[NON FIGÉ — ORIENTATION FAVORABLE]

Le PD n'est pas indispensable, mais il paraît suffisamment peu complexe pour être intéressant dès la V1 si le BOM et le routage restent raisonnables.

Cible envisagée :

```text
source USB-C
   |
   +--> fallback 5 V obligatoire
   |
   +--> si PD disponible : demande 9 V
```

Pourquoi **9 V** :

- compatible avec les principaux chargeurs candidats ;
- réduit le courant dans le câble et le connecteur à puissance équivalente ;
- réduit les pertes I²R ;
- donne plus de marge lorsque Q6A + charge batterie consomment simultanément ;
- évite d'approcher les limites de tension des chargeurs 14/18 V.

Ne pas viser 20 V avec BQ25892/BQ25622 dans cette architecture.

## 5.2 Intérêt réel du PD sur le temps de charge

Pour une batterie ~5000 mAh, téléphone éteint :

- un bon 5 V / 3 A fournit déjà ~15 W ;
- le courant de charge batterie sera ensuite limité par le chargeur et la cellule ;
- 9 V PD ne divise donc pas nécessairement le temps de charge par deux.

Le PD devient surtout utile lorsque :

```text
Q6A actif
+
charge batterie rapide
```

La puissance disponible supplémentaire permet de conserver un bon courant de charge tout en alimentant le système.

## 5.3 Contrôleur PD

[À CHOISIR]

Candidats discutés :

```text
CH224K
CH224A
```

Le CH224A est à regarder en priorité comme solution récente de sink PD simple.

Fonction attendue :

- négocier le profil 9 V lorsqu'il existe ;
- rester à 5 V lorsque la source n'est pas PD ;
- ne pas toucher à D+/D- utilisés par le Q6A ;
- idéalement exposer un statut exploitable par le MCU.

Le contrôleur exact et son statut JLCPCB doivent être vérifiés avant gel du schéma.

## 5.4 Limite de courant avec source 5 V

[À RÉSOUDRE DANS LE SCHÉMA]

Il ne faut pas supposer que toute source 5 V accepte 3 A.

La carte doit avoir un comportement conservateur :

```text
source inconnue / démarrage -> limite faible sûre
source correctement détectée -> courant supérieur autorisé
PD négocié -> limite adaptée au profil obtenu
```

L'utilisation des informations CC/PD et/ou du MCU pour ajuster la limite d'entrée du chargeur reste à définir.

---

# 6. Coulomb counter MCU

## 6.1 Principe

[FIGÉ ARCHITECTURE — VALEURS NON FIGÉES]

Le MCU peut réaliser lui-même le comptage de coulombs :

```text
Q = intégrale de I(t) dt
```

Le shunt doit mesurer le **courant batterie**, pas uniquement le courant SYS.

Schéma fonctionnel :

```text
LiPo+
  |
 shunt
  |
  +----> ampli current-sense bidirectionnel ---> ADC MCU
  |
 BAT du chargeur
  |
 power-path
  |
 SYS
```

Ainsi le MCU mesure réellement :

- courant entrant dans la batterie ;
- courant sortant de la batterie ;
- quasi-zéro batterie lorsque l'USB alimente seul le système.

## 6.2 Exemple de dimensionnement discuté

[EXEMPLE — NON FIGÉ]

Shunt 5 mΩ :

```text
1 A -> 5 mV
5 A -> 25 mV
P @ 5 A = 125 mW
```

Avec un current-sense amplifier gain ~50 et une référence centrée vers VDD/2 :

```text
forte décharge -> tension ADC basse
0 A             -> ~VDD/2
forte charge    -> tension ADC haute
```

L'ampli final doit être :

- bidirectionnel ;
- compatible 3,3 V côté MCU ;
- compatible avec le mode commun batterie 1S ;
- faible offset ;
- faible consommation always-on ;
- suffisamment précis pour que la dérive du coulomb counter reste contrôlable.

Le shunt, son boîtier, sa valeur et l'amplificateur exact restent à choisir.

## 6.3 Correction de dérive

Le MCU ne doit pas faire un simple intégrateur aveugle.

Il devra utiliser :

- tension batterie ;
- température ;
- détection charge complète ;
- périodes de repos pour recalage OCV/SOC ;
- capacité apprise de la cellule ;
- éventuellement historique des cycles.

But : éviter que l'offset ADC/ampli/shunt dérive indéfiniment le SOC estimé.

---

# 7. Thermique et courant de charge

## 7.1 Courant de charge

[NON FIGÉ]

Valeurs de travail proposées :

```text
bring-up initial       : ~1,0 A
validation intermédiaire : ~1,5 A
cible nominale probable : ~2,0 A
test charge rapide      : 2,5 A et plus uniquement après mesures
maximum IC              : dépend du chargeur retenu
```

La limite finale dépend obligatoirement :

- de la cellule réelle ;
- du courant de charge autorisé par son fabricant ;
- du comportement thermique de la PCB ;
- de la coque finale ;
- de la température batterie ;
- de la consommation simultanée du Q6A.

Une cellule 5000 mAh n'implique pas automatiquement qu'une charge 3,5 A est autorisée.

## 7.2 PCB thermique

[À VALIDER]

Une carte 4 couches est fortement envisagée pour :

- plans GND / puissance ;
- boucle buck compacte ;
- réduction EMI ;
- diffusion thermique ;
- vias thermiques sous/près du chargeur ;
- faible impédance des chemins BAT/SYS/VBUS.

Mais 2 couches vs 4 couches n'est pas encore officiellement figé.

Le layout devra suivre le datasheet / layout de référence du chargeur retenu, en particulier :

- condensateurs d'entrée/sortie au plus près ;
- inductance proche du switch ;
- boucle SW minimale ;
- cuivre généreux BAT/SYS/GND ;
- vias thermiques adaptés ;
- séparation des signaux sensibles ADC/I²C du nœud SW.

## 7.3 Mesures obligatoires au bring-up

Mesurer :

- température chargeur ;
- température inductance ;
- température shunt ;
- température connecteur USB-C ;
- température cellule ;
- chute de tension dans les chemins puissance ;
- rendement 5 V et 9 V ;
- comportement avec Q6A actif.

---

# 8. Batterie

## 8.1 Type

[FIGÉ]

Batterie 1S Li-ion/LiPo.

## 8.2 Modèle exact

[NON FIGÉ]

Le modèle AliExpress 307095 ~5000 mAh observé n'est pas validé.

Avant sélection finale, exiger autant que possible :

- capacité crédible ;
- courant de décharge continu ;
- courant de charge recommandé/max ;
- présence et caractéristiques du PCM/protection ;
- dimensions réelles ;
- fabricant ou datasheet exploitable.

Un pack deux fils peut être utilisé avec un **NTC externe collé physiquement à la cellule**.

## 8.3 NTC

[FIGÉ PRINCIPE]

La température de la vraie cellule doit être mesurée par notre carte externe.

Le NTC final et sa courbe restent à choisir en fonction du chargeur.

---

# 9. JLCPCB / stratégie BOM

## 9.1 Règle générale

[FIGÉ]

Avant de choisir une référence, vérifier dans la bibliothèque JLCPCB :

```text
1. Basic
2. Promotional Extended
3. Extended seulement si fonction critique
```

L'objectif est de minimiser le nombre de références Extended différentes.

## 9.2 Composants passifs

À privilégier en Basic lorsque possible :

- 0 Ω ;
- 5,1 kΩ CC ;
- 10 kΩ ;
- 100 kΩ ;
- 100 nF ;
- MLCC usuels ;
- résistances de configuration ;
- pull-up / pull-down ;
- MOSFETs simples si leur RDS(on) est acceptable.

## 9.3 Références discutées

Les références JLC/LCSC citées pendant l'étude doivent être considérées comme **indicatives et à revalider au moment de la commande**, car stock, classification et prix évoluent.

Notamment :

```text
BQ25892 : C165480 lors de la recherche
BQ25622 : disponibilité à revalider
CH224A / CH224K : disponibilité à revalider
```

Ne pas figer un IC uniquement parce qu'il est disponible aujourd'hui si son package ou son architecture est mauvais.

---

# 10. Connectique / interfaces de la carte

## 10.1 USB-C externe

[FIGÉ FONCTION]

```text
VBUS
GND
CC1
CC2
D+
D-
```

Connecteur exact : [À CHOISIR]

## 10.2 Q6A

[FIGÉ FONCTION]

La carte doit fournir :

```text
POWER vers domaine J19/VBATT_PWR
GND
USB D+
USB D-
liaison MCU <-> Q6A à définir
```

Le connecteur J19 lui-même paraît trop gros pour le téléphone final.

V1 : J19 peut rester utilisé pour bring-up.  
Final : connectique plus compacte à définir (pads soudés, FPC, board-to-board, autre).

## 10.3 Batterie

[À CHOISIR]

Prévoir au minimum :

```text
BAT+
BAT-
NTC
```

Le type de connecteur compact n'est pas figé.

## 10.4 Modem

[À DÉFINIR]

La carte power devra au minimum pouvoir fournir/contrôler :

```text
MODEM_PWR
GND
MODEM_EN/PWRKEY selon interface retenue
USB_VBUS modem commutable si nécessaire
```

Le chemin USB D+/D- principal du modem reste à coordonner avec l'architecture Q6A/modem.

## 10.5 MCU debug

[À PRÉVOIR]

Prévoir des pads/testpoints pour :

- programmation ;
- UART debug ;
- I²C ;
- reset ;
- alimentation ;
- GND.

---

# 11. Architecture fonctionnelle V1 proposée

```text
                         USB-C
                           |
             +-------------+-------------+
             |                           |
          D+ / D-                      CC1/CC2
             |                           |
             |                      contrôleur PD
             |                           |
             |                         VBUS
             |                           |
             |                    CHARGEUR + POWER-PATH
             |                    (BQ25892 candidat)
             |                      |           |
             |                     SYS         BAT
             |                      |           |
             |             +--------+           |
             |             |                    |
             |         POWER SWITCHES          SHUNT
             |          |      |      |          |
             |         Q6A    EC25  DISPLAY      |
             |          |                      LiPo 1S
             |          |
             +--------> USB2 Q6A

                  ALWAYS-ON DOMAIN
                         |
                        MCU
                         |
       +-----------------+----------------------+
       |                 |                      |
  I²C chargeur    ADC current-sense      NTC batterie
       |
       +--> télémétrie / contrôle
       |
       +--> MAIN_EN / MODEM_EN / DISPLAY_EN
       |
       +--> informations batterie vers Q6A/Android
```

---

# 12. Points NON FIGÉS à résoudre avant schéma final

## P0 — critiques

```text
[ ] Choisir définitivement le chargeur : BQ25892 vs BQ25622 vs autre candidat JLCPCB
[ ] Vérifier stock/classification JLCPCB au moment du gel BOM
[ ] Choisir si PD est intégré dès la V1 ou seulement footprint optionnel
[ ] Si PD : choisir contrôleur exact et méthode de fallback 5 V
[ ] Définir limite de courant d'entrée sûre avec source non-PD
[ ] Choisir cellule réelle et récupérer ses limites de charge/décharge
[ ] Choisir current-sense amplifier bidirectionnel
[ ] Dimensionner shunt batterie
[ ] Choisir NTC batterie
[ ] Choisir fusible/protection batterie PCB
```

## P1 — power switching

```text
[ ] Définir courant de pointe réel Q6A
[ ] Sélectionner MAIN_PWR switch / MOSFET(s)
[ ] Sélectionner MODEM_PWR switch
[ ] Définir reverse blocking
[ ] Définir coupure VBUS USB modem
[ ] Définir DISPLAY_PWR global ou seulement signaux EN
[ ] Ajouter straps de bypass V1
```

## P1 — MCU

```text
[ ] Choisir MCU always-on
[ ] Définir alimentation always-on
[ ] Définir communication MCU <-> Q6A
[ ] Définir ADC requis / résolution / référence
[ ] Définir interruption chargeur / bouton power / wake
[ ] Définir programmation/debug
```

## P1 — PCB

```text
[ ] 2 couches vs 4 couches
[ ] connecteur USB-C exact
[ ] connecteur batterie exact
[ ] connectique Q6A V1
[ ] interfaces modem / écran
[ ] testpoints
[ ] dimensions mécaniques
[ ] dissipation thermique / zones cuivre
```

---

# 13. Bring-up recommandé

Le bring-up doit être progressif.

## Étape A — carte seule

```text
1. inspection visuelle / court-circuit
2. alimentation labo avec limitation de courant
3. valider domaine always-on
4. valider chargeur sans Q6A
5. valider 5 V non-PD
6. si présent, valider négociation 9 V PD
7. vérifier SYS / BAT
8. vérifier NTC
9. vérifier I²C / télémétrie
10. mesurer températures
```

## Étape B — batterie

```text
1. cellule protégée connue
2. faible courant de charge initial
3. vérifier CC/CV
4. vérifier fin de charge
5. vérifier température cellule
6. vérifier coupure / protections
7. calibrer shunt + ampli
8. vérifier signe charge/décharge
```

## Étape C — Q6A

```text
1. Q6A rework J19 validé
2. power switch MAIN bypassé au besoin
3. démarrer depuis SYS/J19
4. vérifier boot Android
5. USB plug/unplug sans reboot
6. test alimentation USB + charge batterie
7. test batterie seule
8. test Battery Supplement / pics charge
9. test MAIN_EN / shutdown propre
```

## Étape D — modem / écran

```text
1. valider commandes power séparément
2. rechercher tout back-power
3. valider arrêt normal
4. valider hard-off
5. mesurer courant deep-off
```

---

# 14. Cible de la V1

La V1 n'a pas pour objectif d'être immédiatement le PCB final du téléphone.

Elle doit démontrer proprement :

```text
- charge Li-ion 1S sûre
- fonctionnement 5 V universel
- PD 9 V si retenu
- vrai power-path
- alimentation stable Q6A
- USB2 Q6A
- télémétrie chargeur
- coulomb counter MCU
- température batterie
- power gating Q6A / modem / écran
- deep-off
- absence de back-power
```

Une fois ces points validés, une V2 pourra optimiser :

- dimensions ;
- connecteurs ;
- coût ;
- thermique ;
- consommation always-on ;
- intégration mécanique ;
- choix du chargeur final ;
- éventuel USB-C dual-role / OTG plus complet.

---

# 15. Résumé court des décisions

```text
Batterie              : 1S Li-ion/LiPo                     [FIGÉ]
Alimentation Q6A      : via J19 / VBATT_PWR                [FIGÉ]
Charge interne Q6A    : désactivée                         [FIGÉ]
USB data              : USB 2.0 D+/D-                     [FIGÉ]
Fallback charge       : 5 V non-PD obligatoire            [FIGÉ]
Power-path            : oui pour PCB intégrée             [FIGÉ ARCHI]
PD                    : probablement 9 V                  [NON FIGÉ]
Chargeur              : BQ25892 candidat principal        [NON FIGÉ]
BQ25622               : candidat secondaire               [NON FIGÉ]
BQ25638               : non retenu V1 / BGA               [NON RETENU V1]
IP2312                 : prototype/fallback                [NON RETENU PCB principale]
Fuel gauge            : MCU + shunt + current-sense        [FIGÉ ARCHI]
MCU always-on         : oui                                [FIGÉ ARCHI]
Power switch Q6A      : oui                                [FIGÉ ARCHI]
Power switch modem    : oui                                [FIGÉ ARCHI]
Power switch écran    : prévu / à définir précisément     [FIGÉ PRINCIPE]
Courant charge nominal: ~2 A cible initiale               [NON FIGÉ]
PCB 4 couches         : fortement envisagé                [NON FIGÉ]
Cellule exacte        : à choisir                          [NON FIGÉ]
```

---

# 16. Prochaine mission recommandée

Faire un **audit JLCPCB orienté BOM** avant de dessiner le schéma :

```text
1. chargeurs 1S avec power-path disponibles
2. contrôleurs PD sink disponibles
3. current-sense bidirectionnels disponibles
4. MOSFET/load-switchs faible RDS(on) disponibles
5. USB-C receptacle
6. ESD/TVS USB
7. inductance du chargeur
8. fusible / protection batterie
9. NTC
10. passifs Basic
```

Pour chaque référence :

```text
- statut Basic / Promotional Extended / Extended
- stock
- prix
- boîtier
- contraintes de layout
- courant/tension
- consommation quiescente
- rendement
- protections
```

Le schéma doit être figé seulement après cet audit.
