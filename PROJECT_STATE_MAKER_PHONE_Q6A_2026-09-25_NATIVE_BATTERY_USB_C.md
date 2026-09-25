# PROJECT STATE — Q6A batterie 1S externe + J19 + MCU battery manager

**Date :** 2026-09-25  
**Statut :** addendum maître power/battery/USB — architecture batterie désormais retenue.  
**Base Android :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-22_MASTER_ANDROID_BOOT_OK.md`  
**Base écran :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-23_MASTER_SCREEN_PROTO_READY.md`  
**Base STR :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-24_MASTER_STR_FIRST_DEEP_OK.md`

**Règle de priorité :** pour tout ce qui concerne batterie, charge, J19, J21, PM7250B, fuel gauge et architecture USB-C finale, ce document supersède les hypothèses antérieures lorsqu'elles sont contradictoires.

---

# 1. Décision d'architecture retenue

Le projet abandonne l'idée de confier la gestion batterie complète au chargeur/fuel-gauge interne du PM7250B.

La direction retenue est :

```text
                         PCB POWER / BATTERY / USB-C

USB-C extérieur
      |
      +--> contrôleur USB-C / PD / rôle
      |
      +--> chargeur 1S + power-path
      |          |
      |          +--> batterie Li-ion/LiPo 1S protégée
      |                    |
      |                    +--> NTC / mesures tension-courant
      |                    |
      |                    +--> fuel gauge externe / MCU
      |                                 |
      |                                 +--> communication vers Q6A
      |
      +----------------------------------------> J19 Q6A

Q6A :
J19 -> R24 -> VBATT_PWR -> PM7250B -> rails système
```

Le rôle du Q6A est donc volontairement limité :

```text
- J19 sert à alimenter proprement la carte depuis une source 1S
- le PM7250B reste dans le chemin d'alimentation du SoC
- le chargeur PM7250B n'est pas utilisé
- son fuel gauge n'est pas la source de vérité Android
- la gestion batterie réelle est faite sur notre PCB externe
```

Cette architecture évite un chemin inefficace du type :

```text
1S -> boost 12 V -> J21 -> reconversion interne
```

et conserve le chemin énergétique direct 1S vers J19.

Le pack 2S et le chargeur externe haute tension deviennent historiques/fallback uniquement tant que le premier bring-up J19 1S n'a pas été démontré.

---

# 2. Ce que le schéma Q6A V1.21 confirme

Source primaire :

```text
radxa_dragon_q6a_schematic_v1.21.pdf
pages 34 et 35
```

## 2.1 PM7250B

[CONFIRMÉ SCHÉMA]

```text
U26 = Qualcomm PM7250B
page 35 = PM7250B CHG FG
```

Le bloc expose notamment :

```text
USB_IN_x
MID_CHG_x
VSW_CHG_x
PGND_CHG_x
VPH_PWR_x
VBATT_PWR_x
VBATT_SNS_P/M
PACK_SNS_M
ISNS_SMB_P/M
BATT_THERM
BATT_ID
```

Le Q6A possède donc une vraie architecture smartphone 1S native.

La datasheet complète PM7250B n'a pas été récupérée publiquement. Ne pas inventer de limites électriques non sourcées.

## 2.2 J19

[CONFIRMÉ SCHÉMA]

J19 est le point batterie du Q6A.

```text
J19 BAT+
   |
  R24
   |
VBATT_PWR / PM7250B

J19 BAT-
   |
  GND
```

La cible projet est d'utiliser J19 uniquement comme entrée d'alimentation 1S du Q6A.

---

# 3. Configuration Q6A cible

## 3.1 R7 — bypass batteryless

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

```text
R7 = 0 Ω
statut carte = peuplée
```

R7 relie le domaine `VBATT_PWR` au chemin `VPH_PWR_IN` afin de permettre au SOM de fonctionner sans batterie.

[CIBLE]

```text
R7 -> retirée / DNP
```

Une fois R7 retirée, le Q6A ne doit plus dépendre de ce bypass pour son alimentation principale. La source principale devient J19.

Le port USB du Q6A peut toujours être utilisé pour les données si la carte est alimentée depuis J19. En revanche, tant que le chargeur PM7250B n'est pas réactivé, l'USB-C/J21 ne doit pas être considéré comme un moyen de charger la batterie via le Q6A.

## 3.2 R24 — liaison J19 et shunt 2 mΩ

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

```text
R24 = 0,002 Ω 1 %
statut schéma = NC / DNP
statut carte réelle = DNP
```

R24 assure deux fonctions :

```text
1. fermer physiquement le chemin J19 BAT+ -> VBATT_PWR
2. fournir le shunt attendu par l'architecture de mesure PM7250B
```

Même si le projet n'utilise pas le fuel gauge Qualcomm comme source Android, la cible finale reste :

```text
R24 -> 2 mΩ conforme au schéma
```

La perte est négligeable :

```text
3 A -> 6 mV / 18 mW
5 A -> 10 mV / 50 mW
7 A -> 14 mV / 98 mW
```

Un pont `0 Ω` est acceptable uniquement pour un bring-up temporaire afin de vérifier que le Q6A démarre depuis J19. Ce n'est pas la configuration finale recommandée.

## 3.3 R190 — BATT_THERM

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

Le schéma prévoit :

```text
R190 = 100 kΩ vers GND
net = BATT_THERM
```

Note Radxa :

```text
100 kΩ pull-down to GND
= fake good battery temperature
```

La carte réelle inspectée a :

```text
R190 = DNP
```

[CIBLE]

Comme la vraie température batterie sera gérée par notre PCB externe et notre MCU, le PM7250B reçoit une température simulée valide :

```text
BATT_THERM -> 100 kΩ -> GND
```

La résistance peut être déportée dans un boîtier plus facile à souder si le footprint local est trop petit.

## 3.4 R191 — BATT_ID / interdiction de charge PM7250B

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

Le schéma prévoit :

```text
R191 = 10 kΩ vers GND
net = BATT_ID
```

La carte réelle inspectée a :

```text
R191 = DNP
```

Note Radxa :

```text
100 kΩ vers GND
= batterie présente + charge autorisée

2 kΩ à 14 kΩ vers GND
= batterie présente + charge désactivée
```

[CIBLE]

Puisque la charge sera entièrement gérée par notre PCB externe :

```text
BATT_ID -> 10 kΩ -> GND
```

But :

```text
batterie déclarée présente
charge interne PM7250B explicitement désactivée
```

Ne pas utiliser 100 kΩ sur BATT_ID dans l'architecture finale externe.

Comme R191 est très petite, une résistance 10 kΩ déportée en 0603/0805 avec fil fin vers le pad BATT_ID est acceptable et préférable à un rework risqué en 0201.

## 3.5 FB4 — chargeur PM7250B

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

```text
FB4 = ferrite 120 Ω @ HF, 3 A
statut = DNP
```

FB4 relie le domaine d'alimentation externe aux entrées `USB_IN` du chargeur PM7250B.

[CIBLE FINALE]

```text
FB4 -> rester DNP
```

Le chargeur PM7250B n'est pas utilisé dans l'architecture retenue.

---

# 4. Réseau de simulation sense batterie

[CONFIRMÉ SCHÉMA]

Le Q6A prévoit cinq straps 0 Ω DNP :

```text
R185 : VPH_PWR -> VBATT_OPT_ISNS_P
R186 : VPH_PWR -> VBATT_OPT_ISNS_M
R187 : GND     -> VBATT_PACK_SNS_M
R188 : VPH_PWR -> VBATT_VSNS_P
R189 : GND     -> VBATT_VSNS_M
```

[À VÉRIFIER SUR CARTE]

Ils doivent rester DNP pour le bring-up J19 réel, sauf découverte contraire issue du schéma/mesures.

---

# 5. Architecture du PCB power/battery externe

Le PCB externe devient la source de vérité pour toute la gestion batterie.

Il devra au minimum gérer :

```text
- charge Li-ion/LiPo 1S
- power-path chargeur / batterie / système
- protection pack ou coordination avec protection intégrée à la cellule
- mesure tension batterie
- mesure courant charge/décharge
- température batterie via NTC
- estimation SOC / capacité restante
- état charge / décharge / pleine
- détection chargeur
- défauts thermiques et électriques
- communication vers Q6A
```

Le choix exact des IC chargeur, fuel gauge et MCU reste à concevoir.

La protection primaire de la cellule doit rester indépendante des décisions logicielles du MCU.

---

# 6. Intégration MCU -> Q6A -> Android

## 6.1 Principe

Le Q6A ne doit pas utiliser les valeurs du fuel gauge PM7250B comme source batterie principale Android.

La source de vérité devient :

```text
capteurs / fuel gauge externe
          |
         MCU
          |
      bus vers Q6A
          |
   driver Linux battery
          |
  power_supply framework
          |
    Android Health HAL
          |
 icône / pourcentage batterie
```

## 6.2 Données à exposer

Le système externe devra pouvoir fournir au minimum :

```text
capacity / SOC %
voltage_now
current_now
temp
status charge/discharge/full
health
charge_counter si disponible
cycle_count si disponible
présence chargeur
défauts
```

## 6.3 Liaison Q6A <-> MCU

[CIBLE]

I2C est la liaison privilégiée si un bus accessible du Q6A peut être réservé proprement.

Alternatives possibles :

```text
UART
USB interne
SPI si nécessaire
```

Le choix du bus doit tenir compte du suspend-to-RAM : le MCU et le lien doivent permettre au Q6A de récupérer un état batterie cohérent au réveil.

## 6.4 Deux stratégies logicielles possibles

Option A — driver Linux dédié :

```text
MCU protocole simple
      |
driver kernel Q6A
      |
power_supply battery
      |
Android
```

Option B — émulation Smart Battery System / SBS par le MCU :

```text
MCU émule une batterie SBS sur I2C
      |
driver Linux sbs-battery si disponible/activable
      |
power_supply
      |
Android
```

La disponibilité de `CONFIG_BATTERY_SBS` dans le kernel Android Q6A 5.4 doit être vérifiée avant de retenir l'option B.

Ne pas créer une application Android propriétaire uniquement pour afficher le niveau batterie : l'objectif est d'intégrer les informations dans le framework batterie Android natif.

---

# 7. Fuel gauge PM7250B

Le fuel gauge Qualcomm peut éventuellement rester actif en arrière-plan, mais il n'est plus une dépendance du projet.

Il peut être utile pendant le bring-up pour comparer :

```text
mesure tension Qualcomm
mesure courant Qualcomm
SOC Qualcomm

versus

mesures externes de référence
```

Mais Android devra finalement utiliser notre source `power_supply` externe comme batterie principale.

Il est préférable de ne pas supprimer agressivement tous les blocs Qualcomm tant que les dépendances kernel/PMIC ne sont pas comprises. On peut simplement ne pas utiliser leurs valeurs comme source de vérité Android.

---

# 8. Architecture USB-C finale

La précédente hypothèse `USB-C -> J21 -> charge PM7250B` n'est plus la cible principale.

La nouvelle architecture est :

```text
USB-C extérieur
      |
contrôleur CC / PD / rôle
      |
      +--> chargeur/power-path 1S externe
      |           |
      |           +--> batterie
      |           +--> rail système 1S -> J19 Q6A
      |
      +--> USB2 D+/D- -> USB OTG Q6A
      |
      +--> source 5 V contrôlée en mode HOST
```

J21 reste disponible pour :

```text
- alimentation de laboratoire
- récupération / dépannage
- essais spécifiques
```

mais il n'est plus le chemin énergétique normal du téléphone final.

Le petit USB-C soudé sur le Q6A peut rester inutilisé mécaniquement dans le produit final.

---

# 9. USB data / OTG

V1 recommandée : USB2 seulement.

```text
USB-C D+/D-
   -> protections ESD
   -> interface OTG Q6A
```

Fonctions attendues :

```text
ADB
EDL
scrcpy
USB gadget
transfert de fichiers
OTG USB2
clavier/souris/périphériques USB2
```

USB3 SuperSpeed reste une option future et n'est pas nécessaire pour la V1.

Le PCB externe doit empêcher tout conflit de rôle VBUS :

```text
source 5 V active face à un chargeur externe
sink et source simultanés
back-power du Q6A
```

---

# 10. Configuration finale Q6A visée

```text
R7    -> DNP / retirée
R24   -> 2 mΩ 1 %
R190  -> 100 kΩ vers GND
R191  -> 10 kΩ vers GND
FB4   -> DNP
R185..R189 -> DNP si confirmé sur carte
```

Fonction de chaque élément :

```text
R7    : suppression du bypass batteryless
R24   : connexion J19 -> VBATT + shunt conforme à l'architecture Qualcomm
R190  : température batterie simulée valide côté PM7250B
R191  : batterie présente + charge PM7250B désactivée
FB4   : charge PM7250B physiquement non activée
```

---

# 11. Bring-up minimal avant PCB externe final

Premier test : alimentation de laboratoire sur J19.

Ordre :

```text
1. vérifier R185...R189 = DNP
2. garder FB4 DNP
3. retirer R7
4. fermer R24
   - idéalement 2 mΩ
   - 0 Ω possible uniquement pour test temporaire
5. laisser R190/R191 DNP pour le tout premier essai si leur rework est risqué
6. injecter environ 3,8 V sur J19 avec limitation de courant
7. vérifier boot Q6A + Android
8. observer /sys/class/power_supply et dumpsys battery
9. seulement si nécessaire, ajouter R190=100k et R191=10k
```

Le premier essai doit être effectué sans alimentation simultanée via J21/USB-C power.

---

# 12. Décisions gelées au 2026-09-25

| Sujet | État |
|---|---|
| Batterie finale | **1S** |
| J19 utilisé comme alimentation principale Q6A | **OUI** |
| Gestion charge sur PM7250B | **NON** |
| Gestion charge externe | **OUI** |
| Fuel gauge principal Android = PM7250B | **NON** |
| Fuel gauge / mesures externes | **OUI** |
| MCU externe de gestion batterie | **OUI** |
| Intégration native Android via power_supply | **CIBLE** |
| Liaison MCU-Q6A | **I2C privilégié, à valider** |
| R7 | **À RETIRER** |
| R24 final | **2 mΩ** |
| R24 0 Ω | **BRING-UP TEMPORAIRE SEULEMENT** |
| R190 carte actuelle | **DNP confirmé utilisateur** |
| R190 cible | **100 kΩ vers GND** |
| R191 carte actuelle | **DNP confirmé utilisateur** |
| R191 cible | **10 kΩ vers GND** |
| FB4 | **DNP confirmé utilisateur, reste DNP** |
| R185...R189 | **DNP schéma, à vérifier matériel** |
| J21 chemin normal du téléphone final | **NON** |
| USB-C externe unique | **OUI** |
| USB2 suffisant en V1 | **OUI** |
| USB3 obligatoire | **NON** |
| 2S | **ABANDONNÉ SI BRING-UP J19 VALIDÉ** |

---

# 13. Prochain ordre de travail

```text
P0  vérifier physiquement R185...R189
P0  vérifier continuité J19/R24/R7/VBATT_PWR
P0  caractériser précisément le footprint R24 et sélectionner un 2 mΩ compatible
P1  retirer R7
P1  fermer R24
P1  démarrer le Q6A depuis alimentation labo ~3,8 V sur J19
P1  vérifier stabilité, courant, boot Android et power_supply
P1  ajouter R190=100k et R191=10k uniquement si nécessaire / pour config finale
P2  définir le PCB power 1S externe : chargeur + power-path + protection + gauge + MCU
P2  choisir le bus MCU-Q6A
P2  auditer kernel Android pour power_supply custom / sbs-battery
P3  intégrer l'USB-C externe au PCB power
P3  valider suspend/wake avec MCU batterie connecté
```

Ne pas connecter une vraie LiPo avant validation du chemin J19 sur alimentation de laboratoire.