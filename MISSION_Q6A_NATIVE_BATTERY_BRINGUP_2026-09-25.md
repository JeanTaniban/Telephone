# Mission — qualifier la batterie 1S native et la charge PM7250B du Q6A

**Date :** 2026-09-25  
**Carte :** Radxa Dragon Q6A V1.21  
**Objectif :** déterminer expérimentalement si le Q6A peut être utilisé comme base de téléphone avec batterie Li-ion/LiPo 1S directement sur son chemin batterie natif, et si son PM7250B peut assurer charge + power-path + fuel gauge.

Document d'état associé :

```text
PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-25_NATIVE_BATTERY_USB_C.md
```

---

# 1. Règle principale

Ne pas connecter une vraie LiPo tant que le chemin n'a pas été validé sur alimentation de laboratoire.

Ne pas appliquer simultanément plusieurs modifications irréversibles.

Procéder par étapes avec mesures avant/après.

---

# 2. Références schéma

Source primaire :

```text
radxa_dragon_q6a_schematic_v1.21.pdf
page 34 : PM7250_RESERVED / simulation batterie
page 35 : PM7250B CHG FG / batterie / charge
page 28 : DC-IN / USB-C / J21 / ADP12V
```

Composants/nets à suivre :

```text
U26   PM7250B
J19   batterie
J21   alimentation externe / PWR_ON_KEY
R7    0 Ω bypass batteryless
R24   2 mΩ shunt batterie, DNP
R190  100 kΩ BATT_THERM -> GND
R191  10 kΩ BATT_ID -> GND
R185  0 Ω DNP VPH_PWR -> VBATT_OPT_ISNS_P
R186  0 Ω DNP VPH_PWR -> VBATT_OPT_ISNS_M
R187  0 Ω DNP GND -> VBATT_PACK_SNS_M
R188  0 Ω DNP VPH_PWR -> VBATT_VSNS_P
R189  0 Ω DNP GND -> VBATT_VSNS_M
FB4   ferrite 120 Ω HF / 3 A, DNP, ADP12V -> USB_IN PM7250B
```

---

# 3. État matériel déjà constaté

[CONFIRMÉ UTILISATEUR]

```text
R24 : DNP
R7  : peuplée, accessible au dessoudage
R190/R191 : très petites, rework risqué
```

[CONFIRMÉ SCHÉMA]

```text
R190 = 100 kΩ
R191 = 10 kΩ
```

---

# 4. Phase A — inspection sans modification

Carte complètement hors tension :

```text
USB-C Q6A débranché
J21 débranché
USB OTG débranché
aucune batterie
```

## A1 — vérifier les DNP

Inspection loupe/macro :

```text
[ ] R185 DNP
[ ] R186 DNP
[ ] R187 DNP
[ ] R188 DNP
[ ] R189 DNP
[ ] FB4  DNP
```

Documenter par photo si doute.

## A2 — continuités principales

Multimètre en ohmmètre/continuité, carte hors tension.

Vérifier :

```text
[ ] J19- -> GND
[ ] J19+ -> côté batterie de R24
[ ] autre côté R24 -> nœud R7 / VBATT
[ ] R7 -> VPH_PWR_IN
[ ] J21.1 -> ADP12V_IN
[ ] J21.2 -> GND
[ ] J21.3 -> PWR_ON_KEY
```

Ne pas sonder directement les pins PM7250B.

## A3 — BATT_ID

Ne pas dessouder R191 à cette étape.

Si accessible sans risque :

```text
mesurer R191 vers GND = environ 10 kΩ attendu
```

Sinon ne rien faire.

---

# 5. Phase B — préparer le mode batterie sans charge

But : faire fonctionner le Q6A depuis une source 1S simulée **tout en gardant la charge interdite**.

Configuration logique souhaitée :

```text
BATT_THERM = fake good temperature via R190 100 kΩ
BATT_ID    = batterie présente / charge disabled via R191 10 kΩ
```

Donc :

```text
R190 : ne pas toucher
R191 : ne pas toucher
FB4  : rester DNP
```

Avant toute modification : confirmer que R185...R189 sont bien DNP.

## B1 — modification de puissance

Cible probable :

```text
retirer R7
peupler R24 = 2 mΩ 1 % conforme
```

Ne pas utiliser une résistance quelconque de forte valeur à la place du shunt.

Un pont temporaire peut éventuellement servir à un test de conduction très limité, mais ne permet pas de valider la mesure de courant/fuel gauge et n'est pas la configuration cible.

## B2 — première source

Utiliser une alimentation de laboratoire en mode cellule 1S.

Démarrage recommandé :

```text
tension initiale autour de 3,8 V
limitation de courant active
```

Commencer avec une limite prudente et l'augmenter uniquement si le comportement est normal et si le boot nécessite davantage de courant.

Ne jamais dépasser la plage batterie 1S tant que la limite exacte PM7250B n'est pas confirmée par documentation ou mesure.

## B3 — critères de réussite

Observer :

```text
[ ] pas d'échauffement anormal
[ ] courant cohérent
[ ] Q6A démarre
[ ] Android démarre
[ ] tension batterie détectée
[ ] batterie déclarée présente
[ ] charge reste désactivée
```

Collecter par ADB :

```text
dumpsys battery
/sys/class/power_supply/*
uevent/status/voltage/current/capacity si exposés
logs kernel charger/fuel-gauge
```

---

# 6. Phase C — vérifier le fuel gauge

Une fois le boot sur source 1S validé :

Tester plusieurs tensions de laboratoire représentatives sans dépasser la plage sûre :

```text
~3,4 V
~3,7 V
~4,0 V
```

Vérifier :

```text
[ ] voltage_now suit correctement la source
[ ] current_now change avec la charge système
[ ] présence batterie reste stable
[ ] SOC/capacity évolue de façon plausible ou identifier la calibration requise
```

Le SOC peut être faux tant que la configuration battery profile / gauge n'est pas adaptée à la cellule réelle. Ne pas confondre « capteur fonctionne » avec « jauge calibrée ».

---

# 7. Phase D — autoriser la charge

Cette phase n'est autorisée qu'après validation des phases A/B/C.

## D1 — BATT_ID

Cible schéma :

```text
BATT_ID -> 100 kΩ -> GND
```

R191 10 kΩ doit donc être remplacée ou contournée proprement.

Comme le boîtier est très petit, solution privilégiée si le rework direct est trop risqué :

```text
retirer R191
identifier le pad GND par continuité
pad opposé = BATT_ID
fil émaillé fin depuis BATT_ID
résistance 100 kΩ déportée 0603/0805/traversante
retour vers GND accessible
```

Ne pas souder au PM7250B directement.

## D2 — FB4

Après validation du schéma et du driver, peupler la ferrite d'entrée chargeur :

```text
FB4 = ferrite bead 120 Ω HF, 3 A
```

Choisir un composant avec faible résistance DC et courant nominal suffisant, pas une résistance 120 Ω.

## D3 — première charge

Ne pas commencer avec une vraie LiPo non protégée.

Utiliser d'abord une configuration de test permettant de surveiller précisément :

```text
tension VBATT
courant de charge
courant d'entrée
état thermique
tension de fin de charge
état charger Android/Linux
```

Avant vraie cellule, confirmer que la tension de float configurée correspond à la chimie choisie.

---

# 8. Phase E — charge via J21

Le schéma confirme :

```text
J21.1 = ADP12V_IN
J21.2 = GND
J21.3 = PWR_ON_KEY
```

But : valider la charge et le power-path depuis J21 sans utiliser le petit USB-C soudé sur le Q6A.

Tests recommandés après validation batterie :

```text
12 V d'abord
puis 9 V
puis tension intermédiaire si utile
5 V en dernier
```

Pour chaque tension :

```text
[ ] boot possible
[ ] charge possible
[ ] courant d'entrée
[ ] courant batterie
[ ] température
[ ] stabilité à charge CPU/écran/modem
```

La documentation Radxa recommande 12 V ; toute utilisation à plus basse tension doit être considérée expérimentale tant qu'elle n'est pas qualifiée.

---

# 9. Phase F — fallback USB-C non-PD 5 V

But produit : le téléphone doit idéalement accepter un chargeur USB-C 5 V même sans PD.

Deux stratégies à comparer :

```text
F1 : 5 V -> J21
     fonctionne uniquement si le Q6A et la charge restent stables à puissance limitée

F2 : 5 V -> chemin chargeur PM7250B dédié
     batterie alimente le système
     charge lente/degradée
```

F2 n'est pas encore validée et ne doit pas être câblée avant audit complet du chemin `USB_IN`.

Ne pas utiliser un boost 5 -> 12 V comme solution de puissance : il augmente la tension mais pas l'énergie disponible.

---

# 10. Carte USB-C téléphone — cahier des charges provisoire

La carte fille finale regroupera :

```text
USB-C externe unique
├─ CC1/CC2 -> contrôleur USB-C / PD / rôle
├─ VBUS négocié -> protection/switch -> J21 Q6A
├─ D+/D- -> protections ESD -> USB OTG Q6A
└─ 5 V source contrôlée pour mode USB HOST
```

V1 : USB2 uniquement.

Ne pas intégrer le SuperSpeed tant que nécessaire ; cela évite mux SS, routage 90 Ω complexe et contraintes mécaniques supplémentaires.

États à gérer :

```text
1. rien connecté
2. chargeur USB-C / PD -> téléphone sink
3. PC -> téléphone périphérique USB + sink
4. accessoire USB -> téléphone host + source 5 V
```

Interdictions matérielles :

```text
source 5 V et sink VBUS simultanés
back-power Q6A
injection directe d'un VBUS non négocié trop élevé
```

---

# 11. Critère de validation de l'architecture 1S

Le projet abandonne définitivement le 2S lorsque les points suivants sont démontrés :

```text
[ ] boot Q6A fiable depuis J19 / source 1S
[ ] R24 sense exploitable
[ ] batterie détectée correctement
[ ] mesure tension correcte
[ ] mesure courant correcte
[ ] charge activable proprement
[ ] tension de fin de charge maîtrisée
[ ] thermal/BATT_THERM maîtrisé
[ ] fonctionnement charge + système simultané
[ ] charge via J21 validée
[ ] scénario source retirée -> batterie sans reset
[ ] scénario batterie + chargeur -> power-path stable
[ ] aucune surchauffe anormale
```

Après cette validation :

```text
2S -> abandonné
BQ25798 externe -> non nécessaire pour Q6A
chargeur/power-path externe principal -> non nécessaire
batterie finale -> 1S protégée
```

La protection primaire de la cellule reste nécessaire même si le PM7250B gère la charge.

---

# 12. Première action matérielle à faire maintenant

Sans dessouder quoi que ce soit :

```text
1. vérifier R185...R189 = DNP
2. vérifier FB4 = DNP
3. photographier clairement la zone R7/R24 et la zone R190/R191
```

Ensuite seulement établir la procédure précise de retrait R7 + pose R24 pour le premier test alimentation de laboratoire 1S.
