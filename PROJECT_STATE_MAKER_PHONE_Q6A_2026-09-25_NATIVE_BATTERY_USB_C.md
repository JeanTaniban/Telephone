# PROJECT STATE — Q6A batterie native 1S + carte USB-C téléphone

**Date :** 2026-09-25  
**Statut :** addendum maître power/battery/USB — architecture native PM7250B en cours de qualification matérielle.  
**Base Android :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-22_MASTER_ANDROID_BOOT_OK.md`  
**Base écran :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-23_MASTER_SCREEN_PROTO_READY.md`  
**Base STR :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-24_MASTER_STR_FIRST_DEEP_OK.md`

**Règle de priorité :** pour tout ce qui concerne batterie, charge, J19, J21, PM7250B, architecture USB-C finale et abandon potentiel du pack 2S, ce document supersède les hypothèses antérieures lorsqu'elles sont contradictoires.

---

# 1. Décision d'architecture

## 1.1 Direction retenue

[CIBLE CONDITIONNELLE]

Si le chemin batterie natif du Q6A est validé sur matériel, le projet **abandonne l'architecture batterie 2S** et passe à une batterie **Li-ion/LiPo 1S** exploitée directement par le PMIC/chargeur Qualcomm du Q6A.

Architecture visée :

```text
batterie 1S protégée
      |
      v
     J19
      |
   shunt R24
      |
  PM7250B
  ├─ charge
  ├─ power-path
  ├─ mesure tension/courant
  ├─ BATT_THERM
  ├─ BATT_ID
  └─ fuel gauge
      |
    VPH_PWR
      |
     Q6A
```

Le pack 2S + chargeur/power-path externe reste uniquement un **fallback** tant que cette qualification n'est pas terminée.

---

# 2. Ce que le schéma Q6A V1.21 confirme

Source primaire :

```text
radxa_dragon_q6a_schematic_v1.21.pdf
pages 34 et 35
```

## 2.1 PMIC / chargeur principal

[CONFIRMÉ SCHÉMA]

```text
U26 = Qualcomm PM7250B
page 35 : PM7250B CHG FG
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
```

Le Q6A possède donc le matériel nécessaire à un vrai chemin batterie/charge et non de simples pads d'alimentation.

Les informations publiques Qualcomm identifient le PM7250B comme PMIC avec chargeur batterie et fuel gauge. La datasheet complète Qualcomm n'a pas été récupérée publiquement ; ne pas inventer de limites électriques non sourcées.

## 2.2 Pads batterie J19

[CONFIRMÉ SCHÉMA]

J19 est connecté au chemin batterie via R24.

```text
J19 BAT+
  |
 R24
  |
VBATT / PM7250B
```

Le retour batterie va au GND.

Le premier bring-up devra utiliser une alimentation de laboratoire simulant une cellule 1S avant toute vraie LiPo.

---

# 3. Configuration actuelle « SOM sans batterie »

Le Q6A V1.21 est volontairement assemblé pour fonctionner sans vraie batterie.

## 3.1 R24 — shunt batterie

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

```text
R24 = 0,002 Ω 1 %
statut schéma = NC / DNP
statut carte réelle = DNP confirmé
```

R24 se trouve entre les deux points de sense `VBATT_VSNS_P` et `VBATT_VSNS_M`.

Fonction : shunt de mesure de courant batterie et liaison physique du pad BAT+ au domaine VBATT.

Pour une configuration batterie réelle, il devra probablement être peuplé par le shunt spécifié, sous réserve de validation finale du chemin.

## 3.2 R7 — bypass batteryless

[CONFIRMÉ SCHÉMA + INSPECTION UTILISATEUR]

```text
R7 = 0 Ω 1 %
statut carte réelle = peuplé et accessible au dessoudage
```

R7 relie directement le nœud côté PM7250B/VBATT au rail :

```text
VPH_PWR_IN
```

Cette liaison contourne le fonctionnement normal d'une vraie batterie.

[CIBLE]

Pour un mode batterie réel, R7 est un candidat fort à **retirer**. Ne pas le retirer avant la procédure de bring-up dédiée.

## 3.3 R190 — température batterie simulée

[CONFIRMÉ SCHÉMA]

```text
R190 = 100 kΩ vers GND
net = BATT_THERM
```

Note Radxa :

```text
100 kΩ pull-down to GND
= fake good battery temperature
```

Pour les premiers essais, R190 reste en place afin d'éviter une intervention 0201 inutile.

Pour le produit final, une vraie NTC batterie reste préférable si elle peut être raccordée proprement à `BATT_THERM`.

## 3.4 R191 — identification batterie / autorisation charge

[CONFIRMÉ SCHÉMA]

```text
R191 = 10 kΩ vers GND
net = BATT_ID
```

Note Radxa :

```text
100 kΩ vers GND
= batterie présente + charge autorisée

2 kΩ à 14 kΩ vers GND
= batterie présente + charge désactivée
```

Donc l'état actuel `R191 = 10 kΩ` simule volontairement :

```text
batterie présente
charge interdite
```

[CIBLE]

Pour autoriser la charge native :

```text
BATT_ID -> 100 kΩ -> GND
```

R191 est physiquement très petite et le risque de rework est élevé.

Stratégie acceptée :

```text
1. dessouder R191 si nécessaire
2. identifier le pad côté GND par continuité
3. utiliser le pad opposé = BATT_ID
4. souder un fil émaillé très fin sur ce pad
5. déporter une résistance 100 kΩ plus grande (0603/0805/traversante)
6. relier l'autre extrémité à une masse accessible
```

Aucun test-point BATT_ID dédié n'a été identifié dans le schéma.

Ne pas intervenir sur R191 avant d'avoir terminé les tests alimentation batterie avec charge désactivée.

---

# 4. Réseau de simulation sense batterie

[CONFIRMÉ SCHÉMA]

Page 34, le Q6A prévoit cinq straps 0 Ω DNP :

```text
R185 : VPH_PWR -> VBATT_OPT_ISNS_P
R186 : VPH_PWR -> VBATT_OPT_ISNS_M
R187 : GND     -> VBATT_PACK_SNS_M
R188 : VPH_PWR -> VBATT_VSNS_P
R189 : GND     -> VBATT_VSNS_M
```

Tous sont marqués `NC` dans le schéma.

But : permettre des configurations de simulation pour fonctionnement du SOM sans vraie batterie.

[À VÉRIFIER SUR CARTE]

Confirmer physiquement que `R185...R189` sont bien DNP avant passage en vraie batterie.

S'ils sont DNP comme prévu, ne rien modifier.

---

# 5. Entrée de charge du PM7250B

## 5.1 FB4

[CONFIRMÉ SCHÉMA]

```text
FB4 = ferrite bead 120 Ω @ HF, 3 A
statut = NC / DNP
```

Attention : `120R` désigne ici l'impédance HF de la ferrite, pas une résistance DC de 120 Ω.

FB4 est placée sur le chemin `ADP12V` vers les entrées `USB_IN` du PM7250B.

[CIBLE PROBABLE]

Une configuration chargeur native nécessitera probablement de peupler FB4 ou un composant équivalent conforme.

[INTERDIT POUR L'INSTANT]

Ne pas peupler FB4 avant validation complète de :

```text
R7
R24
R185...R189
R190/R191
configuration logicielle charger/fuel gauge
limites de courant/tension
```

---

# 6. SMB1393

[CONFIRMÉ SCHÉMA]

Le schéma contient des signaux et notes relatifs au `SMB1393`, compagnon de charge Qualcomm.

[NON CONFIRMÉ MATÉRIEL]

La présence réelle d'un SMB1393 peuplé sur la Q6A V1.21 n'a pas encore été établie.

Ne pas compter dessus pour l'architecture actuelle tant que le placement/BOM réel n'est pas vérifié.

---

# 7. Sécurité batterie

Même si le PM7250B gère charge, fuel gauge, température et power-path, le projet **ne doit pas considérer le PMIC comme l'unique protection cellule**.

[CIBLE]

Utiliser une cellule/pouch 1S avec protection primaire adaptée ou une protection pack dédiée.

À couvrir indépendamment :

```text
court-circuit pack
surintensité de décharge
surtension cellule
sous-tension profonde
sécurité thermique
```

Le premier bring-up se fait sur alimentation de laboratoire, pas sur une LiPo nue.

---

# 8. Architecture USB-C finale du téléphone

## 8.1 Principe

[CIBLE]

Ne pas déporter mécaniquement le petit USB-C d'alimentation soudé sur la Q6A.

Créer une **carte fille USB-C** positionnée au bord du téléphone.

Cette carte récupère deux interfaces séparées du Q6A :

```text
1. puissance -> J21
2. données   -> port USB 3.1 OTG/HOST bleu du Q6A
```

Le téléphone n'expose qu'un seul USB-C utilisateur.

Architecture :

```text
                       CARTE USB-C TÉLÉPHONE
                  +-----------------------------+
USB-C extérieur --| CC1/CC2 -> contrôleur USB-C|
                  |             / PD            |
                  |                |             |
                  |                +-- VBUS ---- +----> J21 Q6A
                  |                              |
                  | D+/D- -----------------------+----> USB OTG Q6A
                  |                              |
                  | option future USB3 SS       |
                  +-----------------------------+
```

## 8.2 J21

[CONFIRMÉ SCHÉMA]

```text
J21.1 = ADP12V_IN
J21.2 = GND
J21.3 = PWR_ON_KEY
```

J21 est une simple entrée d'alimentation/commande ; **J21 n'est pas lui-même USB-PD**.

Il ne possède pas CC1/CC2.

La négociation PD doit donc être effectuée sur la carte fille USB-C.

## 8.3 PD sur la carte fille

[CIBLE]

La carte fille doit :

```text
- détecter CC1/CC2
- négocier une tension d'entrée compatible avec J21
- commuter/protéger VBUS vers J21
- empêcher tout conflit source/sink
```

Pour le premier prototype, la cible principale reste :

```text
PD 9/12 V -> J21
```

La plage de tension réellement admissible et utilisable de J21 doit être testée expérimentalement ; la documentation Radxa recommande 12 V, même si les composants du front-end acceptent une plage plus large.

---

# 9. Cas d'un chargeur USB-C non-PD 5 V

[À VALIDER]

Un chargeur non-PD fournira typiquement seulement 5 V.

Ne pas supposer que 5 V sur J21 est suffisant pour un téléphone complet.

Problème principal : puissance disponible.

Exemple :

```text
5 V x 3 A = 15 W maximum théorique
```

Un boost 5 -> 12 V n'augmente pas cette puissance et n'est donc pas une solution magique.

Deux voies d'étude restent ouvertes :

```text
A. accepter un mode dégradé à 5 V sur J21
   - puissance limitée
   - charge batterie réduite ou stoppée
   - performances Q6A limitées si nécessaire

B. exploiter directement le chemin charger PM7250B à partir du 5 V USB
   - à étudier
   - ne pas câbler avant validation du schéma et du driver
```

La compatibilité « vieux chargeur 5 V » est souhaitée mais **n'est pas encore démontrée**.

---

# 10. Données USB sur le port USB-C téléphone

## 10.1 V1 recommandée

[CIBLE]

Limiter la première carte fille à **USB 2.0** pour réduire fortement la difficulté de routage :

```text
USB-C D+/D-
   -> protections ESD
   -> interface OTG Q6A
```

Fonctions conservées :

```text
ADB
EDL
scrcpy
USB gadget
transfert de fichiers
OTG USB2
clavier/souris/périphériques USB2
```

## 10.2 USB 3 SuperSpeed

[OPTION FUTURE]

Ajouter le SuperSpeed demanderait :

```text
mux d'orientation USB-C
TX/RX SuperSpeed
90 Ω différentiel
matching
ESD très faible capacité
layout contrôlé
```

Ce n'est pas nécessaire pour la V1 du téléphone.

---

# 11. Mode USB HOST

[CIBLE À CONCEVOIR]

Le même port USB-C doit pouvoir fonctionner :

```text
PC connecté :
- téléphone = sink puissance
- Q6A = périphérique USB
- ADB/EDL possibles

clé USB/accessoire :
- Q6A = host
- téléphone = source 5 V VBUS
```

La carte fille doit donc avoir un vrai contrôle de rôle puissance et empêcher :

```text
source 5 V active face à un chargeur externe
sink PD et source VBUS actives simultanément
back-power du Q6A
```

Le choix du contrôleur USB-C/PD n'est pas encore gelé.

---

# 12. Niveau de difficulté de la carte USB-C

Décision de périmètre :

```text
V1 = USB-C + PD + USB2 + OTG + source 5 V host
```

Difficulté estimée : **faible à moyenne**, nettement inférieure à un interposer DSI si on évite le SuperSpeed.

Les blocs nécessaires seront probablement :

```text
connecteur USB-C
contrôleur CC/PD / role
protections ESD
power switch / MOSFETs
chemin VBUS vers J21
source 5 V pour host
D+/D- vers Q6A OTG
connecteurs/câbles vers J21 et USB Q6A
```

---

# 13. Décisions gelées au 2026-09-25

| Sujet | État |
|---|---|
| PM7250B contient charge/fuel gauge | **CONFIRMÉ** |
| J19 est le chemin batterie natif | **CONFIRMÉ** |
| R24 = 2 mΩ DNP | **CONFIRMÉ** |
| R7 = 0 Ω peuplée | **CONFIRMÉ** |
| R190 = 100 kΩ fake good temperature | **CONFIRMÉ SCHÉMA** |
| R191 = 10 kΩ = charge désactivée | **CONFIRMÉ SCHÉMA** |
| R191 doit viser 100 kΩ pour charge | **CIBLE** |
| BATT_ID possède un gros TP | **NON TROUVÉ** |
| R185...R189 doivent être DNP | **CONFIRMÉ SCHÉMA, À VÉRIFIER MATÉRIEL** |
| FB4 est DNP | **CONFIRMÉ SCHÉMA, À VÉRIFIER MATÉRIEL** |
| batterie finale 1S si bring-up réussi | **DÉCISION CONDITIONNELLE** |
| architecture 2S | **FALLBACK / HISTORIQUE SI 1S VALIDÉ** |
| USB-C externe unique sur carte fille | **CIBLE** |
| puissance carte fille -> J21 | **CIBLE** |
| données carte fille -> OTG Q6A | **CIBLE** |
| USB3 obligatoire en V1 | **NON** |
| USB2 suffisant en V1 | **OUI** |
| J21 est lui-même PD | **NON** |
| compatibilité chargeur 5 V non-PD | **À VALIDER** |

---

# 14. Prochain ordre de travail

```text
P0  vérifier physiquement R185...R189 et FB4
P0  reconstruire exactement les continuités J19/R24/R7/VPH
P0  auditer le driver charger/fuel-gauge Android stock
P0  définir procédure bench 1S sans charge
P1  démarrer Q6A depuis alim labo via J19
P1  valider mesure tension/courant/SOC
P1  seulement ensuite autoriser la charge
P1  qualifier charge depuis ADP12V/J21
P2  mesurer plage réelle J21 (5/6/9/12 V selon protocole)
P2  définir comportement chargeur non-PD 5 V
P2  sélectionner contrôleur USB-C/PD de la carte fille
P3  concevoir carte USB-C V1 en USB2
```

Ne pas connecter une vraie LiPo avant validation P0/P1 sur alimentation de laboratoire.
