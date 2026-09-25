# Mission — qualifier J19 comme alimentation 1S du Q6A avec gestion batterie externe

**Date :** 2026-09-25  
**Carte :** Radxa Dragon Q6A V1.21  
**Objectif :** valider que le Q6A peut être alimenté proprement par une source 1S via J19, tout en déportant la charge, la protection, le fuel gauge, la température et l'intelligence batterie sur notre propre PCB externe.

Document d'état associé :

```text
PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-25_NATIVE_BATTERY_USB_C.md
```

---

# 1. Architecture désormais retenue

Le Q6A ne doit plus être utilisé comme chargeur batterie principal.

Architecture cible :

```text
USB-C extérieur
      |
PCB power / battery externe
├─ contrôleur USB-C / PD / rôle
├─ chargeur Li-ion/LiPo 1S
├─ power-path
├─ protection
├─ fuel gauge / mesure V-I
├─ NTC batterie
└─ MCU
     |
     +--> communication vers Q6A / Android

Batterie / rail système 1S
      |
      +--------------------> J19 Q6A
```

Le Q6A utilise J19 comme entrée batterie 1S uniquement.

Le chargeur et le fuel gauge PM7250B ne sont pas la source principale de gestion batterie.

---

# 2. Règles principales

Ne pas connecter une vraie LiPo tant que J19 n'a pas été validé avec une alimentation de laboratoire.

Ne pas appliquer plusieurs modifications risquées en même temps.

Procéder par étapes avec mesures avant/après.

Le premier objectif est uniquement :

```text
faire démarrer le Q6A depuis J19
```

Pas de charge batterie pendant cette première phase.

---

# 3. Références schéma

Source primaire :

```text
radxa_dragon_q6a_schematic_v1.21.pdf
page 34 : BATT_ID / BATT_THERM / sense batterie
page 35 : PM7250B CHG FG / J19 / R7 / R24 / FB4
page 28 : DC-IN / USB-C / J21
```

Composants/nets principaux :

```text
U26   PM7250B
J19   entrée batterie 1S
R7    0 Ω bypass batteryless, peuplée
R24   2 mΩ shunt / liaison J19 -> VBATT, DNP
R190  100 kΩ BATT_THERM -> GND, DNP sur carte réelle
R191  10 kΩ BATT_ID -> GND, DNP sur carte réelle
R185  0 Ω DNP VPH_PWR -> VBATT_OPT_ISNS_P
R186  0 Ω DNP VPH_PWR -> VBATT_OPT_ISNS_M
R187  0 Ω DNP GND -> VBATT_PACK_SNS_M
R188  0 Ω DNP VPH_PWR -> VBATT_VSNS_P
R189  0 Ω DNP GND -> VBATT_VSNS_M
FB4   ferrite 120 Ω HF / 3 A, DNP
```

---

# 4. État matériel déjà constaté

[CONFIRMÉ UTILISATEUR]

```text
R7   : peuplée, accessible au dessoudage
R24  : DNP
R190 : DNP
R191 : DNP
FB4  : DNP
```

[À VÉRIFIER]

```text
R185 : DNP ?
R186 : DNP ?
R187 : DNP ?
R188 : DNP ?
R189 : DNP ?
```

---

# 5. Configuration finale Q6A visée

```text
R7    -> retirée / DNP
R24   -> 2 mΩ 1 %
R190  -> 100 kΩ vers GND
R191  -> 10 kΩ vers GND
FB4   -> DNP
R185..R189 -> DNP si inspection confirme le schéma
```

Rôle :

```text
R7    : suppression du bypass de fonctionnement sans batterie
R24   : connexion électrique J19 -> VBATT_PWR + shunt conforme au design Qualcomm
R190  : simule une température batterie valide pour le PM7250B
R191  : signale batterie présente mais interdit la charge PM7250B
FB4   : maintient le chargeur PM7250B hors du chemin de charge
```

---

# 6. Phase A — inspection sans modification

Carte complètement hors tension :

```text
USB-C power débranché
J21 débranché
USB OTG débranché si possible
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
[ ] R24  DNP
[ ] R190 DNP
[ ] R191 DNP
[ ] FB4  DNP
```

## A2 — continuités principales

Multimètre, carte hors tension :

```text
[ ] J19- -> GND
[ ] J19+ -> pad côté batterie de R24
[ ] autre pad R24 -> domaine VBATT / R7
[ ] R7 -> VPH_PWR_IN
```

Le but est de reconstruire exactement le chemin avant soudure.

Ne pas sonder directement les billes/pins du PM7250B.

---

# 7. Phase B — premier bring-up J19 minimal

But : démontrer que J19 peut alimenter le Q6A.

## B1 — retirer R7

Retirer :

```text
R7 = 0 Ω
```

Conséquence attendue : le bypass `VPH_PWR_IN -> VBATT_PWR` n'alimente plus directement le domaine batterie.

Pour le premier test, ne pas brancher simultanément une source d'alimentation sur J21 / USB-C power.

Le port USB de données pourra être réintroduit plus tard une fois le chemin d'alimentation maîtrisé.

## B2 — fermer R24

Configuration cible finale :

```text
R24 = 2 mΩ 1 %
```

R24 est indispensable pour fermer :

```text
J19 BAT+ -> VBATT_PWR
```

Si le composant 2 mΩ exact n'est pas encore disponible, un pont `0 Ω` peut servir à un test temporaire de boot.

Dans ce cas :

```text
- ne pas valider le fuel gauge PM7250B
- ne pas considérer la configuration comme finale
- remplacer ensuite par 2 mΩ
```

Le 2 mΩ final n'est pas une source de perte significative et conserve l'architecture électrique attendue par le PM7250B.

## B3 — source laboratoire

Utiliser une alimentation de laboratoire simulant une cellule 1S.

Point de départ :

```text
~3,8 V
limitation de courant active
```

Ne pas connecter de vraie LiPo à ce stade.

## B4 — critères de réussite

Observer :

```text
[ ] aucun échauffement anormal
[ ] courant cohérent
[ ] Q6A démarre
[ ] Android démarre
[ ] aucun reset spontané
[ ] stabilité en veille puis réveil à tester ensuite
```

Collecter :

```bash
dumpsys battery
ls -l /sys/class/power_supply
```

et, si disponibles :

```text
voltage_now
current_now
capacity
status
temp
uevent
```

Le résultat du fuel gauge Qualcomm est informatif uniquement ; il n'est plus un critère de validation de l'architecture finale.

---

# 8. Phase C — ajouter les états batterie simulés côté PM7250B

Cette phase est utile si le PM7250B ou Android réagit mal avec `BATT_THERM` et `BATT_ID` flottants, ou pour préparer la configuration finale.

## C1 — R190

Ajouter :

```text
BATT_THERM -> 100 kΩ -> GND
```

But : simuler une température batterie valide côté PM7250B.

Comme le footprint est petit, privilégier si nécessaire :

```text
pad signal R190 -> fil fin -> 100 kΩ 0603/0805 -> GND accessible
```

La vraie température sera mesurée sur le PCB externe, pas par ce réseau simulé.

## C2 — R191

Ajouter :

```text
BATT_ID -> 10 kΩ -> GND
```

Le schéma Radxa indique que 2 kΩ à 14 kΩ signifie :

```text
batterie présente
charge désactivée
```

C'est exactement le comportement voulu puisque la charge est gérée sur notre PCB externe.

Ne pas mettre 100 kΩ en configuration finale externe : cela demanderait au PM7250B d'autoriser sa propre charge.

---

# 9. Phase D — vérifier fonctionnement simultané avec USB data

Une fois le boot J19 stable :

```text
1. Q6A alimenté uniquement depuis J19
2. connecter le chemin USB data au PC
3. vérifier ADB / scrcpy / transfert
4. vérifier qu'aucun chemin power indésirable ne backfeed la carte
```

Si le câble/port utilisé injecte aussi du VBUS, mesurer les tensions et courants avant de considérer ce mode comme sûr.

L'objectif final est de séparer proprement :

```text
alimentation système -> J19
USB data -> interface OTG Q6A
```

---

# 10. PCB externe — fonctions à concevoir

Le futur PCB power/battery devra fournir :

```text
chargeur 1S
power-path
protection pack
mesure tension
mesure courant charge/décharge
NTC batterie
SOC / capacité restante
état charge / discharge / full
présence chargeur
défauts
MCU
USB-C / PD / rôle
```

La protection primaire de la cellule doit fonctionner même si le MCU ou Android est planté.

---

# 11. Communication MCU -> Q6A

Objectif : Android affiche les informations de notre système externe comme une batterie native.

Architecture cible :

```text
fuel gauge / capteurs externes
           |
          MCU
           |
          I2C
           |
          Q6A
           |
      driver Linux
           |
      power_supply
           |
   Android Health HAL
           |
  pourcentage / température / état
```

Données minimales :

```text
SOC %
tension
courant
température
status charge/discharge/full
health
présence chargeur
```

Données souhaitables :

```text
charge_counter
cycle_count
capacité restante
capacité pleine
fault flags
```

---

# 12. Intégration Linux / Android à étudier

Deux solutions principales :

## Solution A — driver dédié

```text
MCU avec protocole simple
       |
driver kernel Q6A
       |
power_supply
       |
Android
```

## Solution B — MCU compatible SBS

```text
MCU émule une Smart Battery SBS sur I2C
       |
driver Linux sbs-battery
       |
power_supply
       |
Android
```

Avant de choisir la solution B :

```text
[ ] vérifier CONFIG_BATTERY_SBS dans le kernel Q6A
[ ] vérifier bus I2C libre et accessible
[ ] vérifier comportement suspend/resume
```

Ne pas développer une simple application Android d'affichage batterie si l'intégration native `power_supply` est possible.

---

# 13. USB-C final

Le chemin d'alimentation final n'est plus :

```text
USB-C -> J21 -> PM7250B charger
```

La cible devient :

```text
USB-C
  |
contrôleur CC / PD
  |
chargeur + power-path externe
  |
  +--> batterie
  |
  +--> rail 1S système -> J19

USB-C D+/D-
  |
USB OTG Q6A
```

J21 reste disponible pour dépannage, labo ou scénarios spécifiques, mais n'est plus le chemin power normal du téléphone final.

---

# 14. Critères de validation J19

L'architecture peut être considérée viable lorsque :

```text
[ ] R7 retirée sans anomalie
[ ] J19 alimente le Q6A via R24
[ ] boot Android fiable à ~3,8 V
[ ] fonctionnement stable sur plage 1S à qualifier
[ ] pas de surchauffe anormale
[ ] USB data fonctionne pendant alimentation J19
[ ] suspend/resume fonctionne sur alimentation J19
[ ] R190=100k et R191=10k donnent un état PM7250B stable
[ ] FB4 reste DNP
[ ] aucun backfeed dangereux depuis USB/J21
```

Le fuel gauge PM7250B n'a pas besoin d'être précis pour valider l'architecture.

---

# 15. Première action matérielle à faire maintenant

Sans modifier R190/R191 pour l'instant :

```text
1. vérifier R185...R189 = DNP
2. relever les dimensions / footprint de R24
3. vérifier les continuités J19 -> R24 -> VBATT -> R7
4. préparer retrait R7
5. préparer R24 = 2 mΩ ou pont 0 Ω temporaire
6. faire le premier test alimentation labo ~3,8 V sur J19
```

Ne pas brancher une vraie LiPo avant ce test.