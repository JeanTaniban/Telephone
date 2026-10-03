# GUIDE — Validation pré-fabrication carte Power/MCU V1 — 2026-10-03

**Référence électrique :** `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md`  
**État projet :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md`  
**Décisions consolidées :** `DECISIONS_ARCHITECTURE_POWER_MCU_2026-10-03.md`

But : valider uniquement les risques capables d'imposer une modification PCB avant de figer schéma/placement V0.5. Les fonctions Android/radio pouvant être corrigées après fabrication ne doivent pas bloquer indéfiniment la carte.

---

# 1. Q6A J19 / alimentation 1S

Configuration finale visée :

```text
R7    DNP
R24   2 mOhm 1 %
R190  100 kOhm -> GND
R191  10 kOhm -> GND
FB4   DNP
R185..R189 DNP confirmés
```

Essais alimentation labo :

```text
4.20 V
3.80 V
3.40 V
3.00 V
```

À chaque point : boot, idle, charge CPU, courant, température ; au moins 3.8/3.4 V en vrai STR.

PASS : aucun reset spontané, pas d'échauffement anormal, courant cohérent.

---

# 2. PWR_ON_KEY / SLEEP_REQ / HEARTBEAT

## 2.1 PWR_ON_KEY

Prototype S8050 depuis RP2040 ou générateur open-collector.

```text
[ ] J19 alimenté + Q6A arrêté -> pulse -> cold boot
[ ] Q6A deep -> pulse -> wake
[ ] Q6A RUN -> pulse court -> événement Power attendu
[ ] 10 cold boots consécutifs
[ ] 10 wakes deep consécutifs
```

Le test doit être réalisé avec l'architecture J19 finale ; la réussite du bouton Power avec l'ancienne alimentation Q6A ne suffit pas.

## 2.2 SLEEP_REQ

GPIO58 doit déclencher la séquence logicielle de suspend, pas seulement éteindre l'écran.

Vérifier :

```text
mem_sleep = deep
OTG externe -> none si requis
USB EC25 suspend
Wi-Fi/wake sources traités
entrée effective en deep
```

## 2.3 HEARTBEAT

GPIO59 configuré en sortie Q6A. Vérifier réception stable sur GP27 avec le réseau 100 k série / 1 M pulldown.

Le test doit aussi prouver que l'arrêt volontaire du heartbeat en deep n'entraîne aucune coupure MAIN_PWR et qu'un heartbeat absent au boot/shutdown ne génère pas de faux FAULT.

---

# 3. MAIN_PWR / MODEM_PWR hold RC

Prototype ou simulation + mesure réelle dès que possible :

```text
AO3400A C20917
10 kOhm série GPIO
1 MOhm pulldown
1 uF gate-GND
100 kOhm P-gate-SYS
```

## 3.1 Reset MCU alors que le MCU reste alimenté

Mesurer :

```text
[ ] rail OFF par défaut
[ ] GPIO HIGH -> ON franc
[ ] GPIO LOW -> OFF rapide
[ ] GPIO Hi-Z -> hold ~0.7..1.5 s cible
[ ] watchdog/reset RP2040 -> rail ne chute pas
[ ] firmware réaffirme GP10/GP29 avant expiration du hold
```

Tester MAIN et MODEM séparément.

## 3.2 Storage OFF / MCU réellement désalimenté

Le cas `STORAGE_SW` ouvert est électriquement différent d'un simple reset : le 3V3 du RP tombe alors que `C_HOLD` peut encore être chargé.

Vérifier à l'oscilloscope et en courant :

```text
[ ] aucune injection/back-power problématique vers le RP2040 par GP10/GP29
[ ] pas de latch-up ou redémarrage parasite du MCU
[ ] MAIN/MODEM finissent bien OFF
[ ] courant de stockage retombe à la valeur attendue
```

Le hold exact d'une seconde n'est pas requis pendant Storage OFF ; l'objectif est un comportement sûr et déterministe.

---

# 4. EC25 alimentation depuis SYS

Le modem est alimenté par `SYS -> MODEM_PWR -> carrier BAT`.

Banc :

```text
[ ] BAT carrier mesuré à batterie/SYS haut
[ ] BAT carrier toujours <4.30 V
[ ] démarrage EC25 fiable
[ ] pic TX/LTE observé à l'oscilloscope
[ ] VSYS ne s'effondre pas sous Q6A + EC25
[ ] pas de reset Q6A
[ ] pas de reset EC25
```

Tester au moins autour de 4.2 / 3.8 / 3.4 V batterie.

Le pack/PCM/câblage retenu doit avoir une marge de courant suffisante pour les pics combinés.

Ajouter un essai de pire cas système :

```text
Q6A en charge CPU + EC25 en émission LTE + MCU/annexe active
```

Observer `BAT`, `SYS`, `MAIN_PWR_OUT`, `MODEM_PWR_OUT` et la température du BQ/cuivres/connectique. Répéter batterie seule puis USB 5 V branché afin de vérifier correctement le power-path et le complément batterie.

---

# 5. EC25 USB / deep sleep

Câblage direct :

```text
Q6A host VBUS/D+/D-/GND -> EC25 VBUS/DP/DN/GND
```

Avec MODEM_PWR ON :

```text
AT+QSCLK=1
DTR HIGH
host Q6A suspend USB
```

Vérifier consommation et état modem.

Deux tests critiques :

```text
[ ] deep modem avec VBUS USB toujours présent
[ ] MODEM_PWR OFF + VBUS présent -> pas de back-power significatif
```

Si le premier échoue, prévoir coupure du fil VBUS dans le harnais ou révision ultérieure.

---

# 6. USB-C extérieur

Fonction : sink/device uniquement.

```text
CC1 5.1 k -> GND
CC2 5.1 k -> GND
D+/D- -> Q6A OTG
VBUS -> BQ
VBUS -> Schottky -> Q6A OTG VBUS
```

Tests avant fabrication si montage filaire possible :

```text
[ ] PC détecte Q6A en ADB/device
[ ] EDL reste possible
[ ] chargeur USB-C C-to-C fournit 5 V
[ ] aucune source host annoncée par le téléphone
[ ] MAIN_PWR OFF + USB-C branché -> pas de back-power Q6A problématique
```

Réseau CC fusionné : valider sur trois sources si disponibles : Default, 1.5 A, 3 A. Toute mesure ambiguë doit retomber en Default.

---

# 7. BQ25628E / charge

Avant layout final : revue datasheet + application layout.

```text
[ ] RILIM 5.6 k
[ ] CE -> GND
[ ] NTC réseau 5.1 k / 30 k / 103AT
[ ] passifs BTST/REGN/VBUS/PMID/SYS/BAT corrects
[ ] VREG firmware 4.15 V de départ
[ ] VSYSMIN 3.84 V
```

Layout :

```text
[ ] PMID caps immédiatement au BQ
[ ] boucle SW minimale
[ ] SYS caps immédiatement au BQ
[ ] SW petit et éloigné TS/CC/I2C
[ ] vias thermiques/GND suffisants
[ ] pistes BAT/SYS larges
[ ] changements de couche puissance avec vias multiples
```

Après fabrication : montée progressive 0.5 / 1 / 1.5 / 2 A avec thermique.

---

# 8. TPS610995 / Storage OFF

Revue datasheet puis test prototype :

```text
[ ] EN LOW -> shutdown/isolation attendu
[ ] MCU_VSYS tombe réellement
[ ] courant résiduel stockage acceptable
[ ] USB branché n'alimente pas le RP par un autre chemin
```

Interfaces à examiner pour back-power :

```text
BQ I2C/INT
CC_SENSE
Q6A SLEEP_REQ/HEARTBEAT/SBS
EC25 level-shifter
PWRKEY drivers
MAIN/MODEM C_HOLD via GP10/GP29
```

---

# 9. SBS / I2C Q6A

Avant peuplement des pull-up :

```text
[ ] mesurer SDA/SCL Q6A
[ ] identifier les pull-up déjà présents
[ ] si nécessaires seulement : peupler 4.7 k vers Q6A_3V3
```

Ne pas utiliser 3V3_MCU comme source de pull-up SBS.

---

# 10. RP2040-Tiny / mécanique

```text
[ ] module réel mesuré
[ ] footprint imprimé 1:1
[ ] orientation validée
[ ] FPC accessible
[ ] aucun via/testpoint sous zone dangereuse
[ ] face BOTTOM sans collision
```

GP28 doit rester accessible via `TP_GP28_3V3`; l'étage RESET_N open-drain est DNP.

---

# 11. USB2 PCB

Seul USB-C -> Q6A traverse la carte Power.

```text
[ ] stack-up JLC choisi avant calcul largeur/écartement
[ ] Zdiff ~90 Ohm
[ ] D+/D- même couche
[ ] plan GND continu
[ ] pas de stub
[ ] peu de vias
[ ] ESD près du connecteur
```

Q6A -> EC25 reste un faisceau direct court.

---

# 12. Revue finale avant Gerber

```text
[ ] schéma conforme au CDC 2026-10-03
[ ] pinout RP2040 conforme
[ ] MAIN/MODEM RC = 1 MOhm + 1 uF
[ ] GPIO59 = HEARTBEAT, pas WAKE
[ ] GP8 = PWR_ON_KEY
[ ] GP26 = CC_SENSE fusionné
[ ] GP28 = réserve
[ ] GP29 = MODEM_PWR
[ ] MODEM_PWR pads dimensionnés pour courant modem
[ ] testpoints obligatoires présents
[ ] ERC propre
[ ] DRC propre
[ ] BOM régénérée depuis schéma
[ ] références JLC Basic/Extended revalidées
[ ] PnP/orientation vérifiés
[ ] polarités USB-C/TVS/diode/MOSFET vérifiées
[ ] PCB <=70 x 25 mm
```

---

# 13. Non-bloquants PCB

```text
- data Android complète
- SMS complet
- VoLTE / IMS
- audio appel final
- démarrage autonome RIL final
- précision SOC finale
```

Ces points peuvent continuer après commande si les interfaces électriques sont correctement prévues.
