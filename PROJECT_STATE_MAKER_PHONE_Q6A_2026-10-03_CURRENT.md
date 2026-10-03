# PROJECT STATE — Maker Phone Q6A — CURRENT

**Date :** 2026-10-03  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **référence courante pour Power/MCU/modem/suspend et préparation schéma V0.5**

## Priorité documentaire

Pour Power / batterie / MCU / modem / suspend, utiliser cet ordre :

```text
1. CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md
2. PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md
3. MISSION_MCU_SUPERVISION_V2_2026-10-03.md
4. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
5. PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md
6. documents STR/écran spécialisés
7. documents antérieurs = historique
```

Ne pas réintroduire automatiquement : USB-C host/DRP, EC25 alimenté principalement par VBUS Q6A, GPIO59 comme WAKE final, CC1/CC2 sur deux ADC, RESET_N EC25 dédié, architecture audio Minimal autonome.

---

# 1. Plateforme

```text
SBC            : Radxa Dragon Q6A V1.21
SoC            : Qualcomm QCS6490
Android        : Android 15 Radxa 20260630-b1, boot validé
Stockage       : eMMC 64 GB
Modem          : Quectel EC25-EUXGA
MCU V1         : Waveshare RP2040-Tiny
Batterie cible : Li-ion/LiPo 1S ~5000 mAh protégée, 4.20 V max
```

Le Q6A fonctionne sur Android réel. Le STR profond a déjà été obtenu expérimentalement lorsque l'USB OTG externe ne bloque pas SystemSuspend ; le port externe doit donc être mis en `none` avant deep si nécessaire.

---

# 2. Architecture Power courante

```text
USB-C extérieur 5 V / device
    |
    +--> BQ25628E -> SYS -> batterie 1S
                         |
                         +--> MAIN_PWR -> Q6A J19
                         +--> MODEM_PWR -> EC25 carrier BAT
                         +--> ANNEXE1
                         +--> ANNEXE2
                         +--> TPS610995 -> MCU_VSYS -> RP2040-Tiny

Q6A USB2 host #1 -> EC25 VBUS/D+/D-/GND
Q6A USB2 host #2 -> caméra USB future
Q6A USB2 host #3 -> réserve / futur USB-A amovible

USB-C D+/D- -> Q6A OTG en DEVICE uniquement
```

Décisions principales :

```text
- BQ25628E, charge 1S jusqu'à 2 A.
- USB-C 5 V seulement ; charge + ADB/EDL/MTP.
- aucun USB-C host/DRP/PD V1.
- MAIN_PWR et MODEM_PWR OFF par défaut.
- MAIN_PWR et MODEM_PWR maintenus ~1 s pendant reboot MCU par RC sur AO3400A.
- stockage : TPS610995 coupé par STORAGE_SW.
- EC25 puissance principale depuis SYS/MODEM_PWR, pas depuis USB VBUS.
- USB EC25 direct Q6A <-> carrier, hors PCB Power.
- deux ANNEXE simples OFF par défaut.
- PCB 4 couches, <=70 x 25 mm.
- RP2040-Tiny face BOTTOM, FPC accessible.
```

---

# 3. Q6A J19 / batterie native

Configuration Q6A cible :

```text
R7    -> retiré / DNP
R24   -> 2 mOhm 1 %
R190  -> 100 kOhm vers GND
R191  -> 10 kOhm vers GND
FB4   -> DNP
R185..R189 -> DNP si inspection physique confirme
```

J19 est utilisé comme entrée batterie 1S. Le chargeur externe BQ25628E et le power-path vivent sur notre carte.

Validation au banc : 4.2 / 3.8 / 3.4 / 3.0 V avec courant, température, boot, charge CPU et vrai deep.

---

# 4. Commande Q6A par le MCU

Architecture finale :

```text
RP GP8  -> S8050 -> vrai Q6A PWR_ON_KEY
RP GP9  -> Q6A GPIO58 = SLEEP_REQ
Q6A GPIO59 -> RP GP27 = HEARTBEAT
RP GP14 <-> Q6A I2C6_SDA = SBS_SDA
RP GP15 <-> Q6A I2C6_SCL = SBS_SCL
```

Le `PWR_ON_KEY` remplace le rôle de `GPIO59 WAKE` :

```text
cold boot
wake depuis deep
événement Power Android
appui long de recovery
```

`SLEEP_REQ` est conservé car le suspend normal doit exécuter notre procédure propre : USB modem suspend, DTR/QSCLK, OTG externe `none`, Wi-Fi/wake sources, puis `mem_sleep=deep`.

Le heartbeat est un statut logiciel. Il peut s'arrêter volontairement en deep ; son absence ne provoque jamais une coupure immédiate.

---

# 5. MAIN_PWR / MODEM_PWR et reboot MCU

Chaque rail critique :

```text
JMTQ55P02A high-side
AO3400A C20917 comme driver gate
10 kOhm GPIO -> gate AO
1 MOhm gate AO -> GND
1 uF gate AO -> GND
100 kOhm P-gate -> SYS
```

Effet :

```text
MCU absent longtemps -> rail OFF
MCU reboot court     -> rail reste ON ~1 s
GPIO LOW volontaire  -> rail coupe rapidement
```

Au boot, le firmware doit lire GP10/GP29 immédiatement et réaffirmer HIGH avant expiration du RC si les rails étaient déjà actifs.

---

# 6. USB-C extérieur

Fonction définitive :

```text
charge 5 V
ADB
EDL
MTP / device
```

```text
CC1 -> 5.1 kOhm -> GND
CC2 -> 5.1 kOhm -> GND
CC1 + CC2 -> réseau 470 kOhm + 470 kOhm -> CC_SENSE GP26
CC_SENSE -> 100 nF -> GND
```

Le courant source Type-C est lu sur un seul ADC. Toute mesure ambiguë reste en courant Default/conservateur.

USB-C VBUS alimente le BQ et rejoint Q6A OTG VBUS via une Schottky Basic B5819W. Vérifier absence de back-power lorsque MAIN_PWR est OFF.

---

# 7. Ports USB Q6A

Hors port OTG bleu utilisé pour USB-C device, les trois USB2 host sont affectés ainsi :

```text
HOST #1 -> EC25
HOST #2 -> caméra USB future
HOST #3 -> libre / extension future USB-A amovible
```

Cette répartition permet de garder l'USB-C externe simple et déterministe.

---

# 8. EC25 alimentation / sommeil

Alimentation :

```text
SYS -> MODEM_PWR -> carrier BAT
Q6A USB host -> carrier VBUS/DP/DN/GND
```

Le domaine EC25 est 3.3–4.3 V ; cible 3.8 V. Firmware de départ : autoriser MODEM_PWR environ entre 3.35 et 4.25 V, avec mesure réelle obligatoire pour garantir BAT <4.30 V dans tous les états.

Pas de switch VBUS EC25 V1. Il faut donc vérifier que le host Q6A suspend réellement l'USB en deep. Si ce point échoue, le câble direct pourra être modifié pour couper VBUS sans refaire immédiatement la carte Power.

Ordre de recovery modem :

```text
AT propre -> PWRKEY -> MODEM_PWR power-cycle
```

`RESET_N` n'est plus une fonction normale.

---

# 9. EC25 <-> MCU

```text
GP0 -> EC25 RXD via SN74AVC4T245
GP1 <- EC25 TXD via SN74AVC4T245
GP2 -> EC25 DTR via SN74AVC4T245
GP3 <- EC25 RI via SN74AVC4T245
GP13 -> EC25 PWRKEY via S8050
GP28 = réserve / TP 3.3 V / option RESET_N open-drain DNP
GP29 -> MODEM_PWR_EN
```

Le SN74AVC4T245 est entièrement occupé par TX/RX/DTR/RI. GP28 expose donc un testpoint 3.3 V direct et un étage open-drain DNP vers le domaine RESET_N/1.8 V, pas un vrai push-pull 1.8 V.

---

# 10. Pinout RP2040-Tiny courant

```text
GP0   -> EC25 RXD / UART TX MCU
GP1   <- EC25 TXD / UART RX MCU
GP2   -> EC25 DTR
GP3   <- EC25 RI
GP4   <-> BQ SDA
GP5   ->  BQ SCL
GP6   <-  BQ INT
GP7   <-  PWR_BUTTON utilisateur
GP8   ->  Q6A PWR_ON_KEY
GP9   ->  Q6A SLEEP_REQ
GP10  ->  MAIN_PWR_EN + RC hold
GP11  ->  ANNEXE1_EN
GP12  ->  ANNEXE2_EN
GP13  ->  EC25 PWRKEY
GP14  <-> Q6A SBS_SDA
GP15  <-> Q6A SBS_SCL
GP26  <-  CC_SENSE fusionné
GP27  <-  Q6A HEARTBEAT / GPIO59
GP28  <-> réserve / TP 3.3 V / option RESET_N OD DNP
GP29  ->  MODEM_PWR_EN + RC hold
```

GP28 reste volontairement la seule réserve pratique.

---

# 11. SBS / batterie Android

Q6A I2C6 :

```text
J20 pin 3 GPIO24 / SDA
J20 pin 5 GPIO25 / SCL
```

Prévoir 2 x 4.7 kOhm vers `Q6A_3V3` **DNP par défaut**. Ne les peupler que si les pull-up Q6A ne sont pas déjà présents/utilisables. Ne pas tirer le bus vers 3V3_MCU pour éviter le back-power Q6A.

Le MCU peut exposer une batterie virtuelle à Android à partir des ADC/états BQ et d'un modèle logiciel. Ce n'est pas un fuel-gauge de précision.

---

# 12. Audio

La V1 n'utilise pas le PCM EC25 ni un codec autonome Minimal.

```text
Téléphonie/media -> Q6A/Android
Q6A HPH_L/R -> oreillette + entrée haute impédance ampli stéréo
ANNEXE -> alim/EN ampli HP
Bluetooth -> audio externe
pas de jack utilisateur
```

Le MCU peut couper l'ampli pendant un appel.

---

# 13. Machine d'états

```text
STORAGE : MCU/Q6A/EC25 OFF
OFF     : MCU ON, MAIN/MODEM OFF
BOOT    : MAIN ON -> PWR_ON_KEY -> attendre heartbeat -> MODEM si VSYS sûr
RUN     : heartbeat actif
DEEP    : MAIN ON ; heartbeat éventuellement absent volontairement ; wake par PWR_ON_KEY
SHUTDOWN: Android + modem arrêtés proprement -> MODEM OFF -> MAIN OFF
FAULT   : récupération graduelle, jamais coupure immédiate sur simple perte heartbeat
```

---

# 14. Gates avant fabrication

```text
[ ] J19 Q6A rework final et caractérisation 1S.
[ ] PWR_ON_KEY : cold boot + wake deep.
[ ] SLEEP_REQ + HEARTBEAT sur GPIO58/59.
[ ] EC25 VIO/TXD/RI mesurés.
[ ] MODEM_PWR depuis SYS : BAT <4.30 V, pics LTE stables.
[ ] USB host Q6A suspend le EC25 en deep avec VBUS présent.
[ ] aucun back-power EC25 lorsque MODEM_PWR OFF.
[ ] aucun back-power Q6A via USB-C lorsque MAIN_PWR OFF.
[ ] TPS610995 Storage OFF / isolation / courant résiduel.
[ ] SBS pull-up vérifiés avant peuplement.
[ ] footprint RP2040-Tiny 1:1 + FPC accessible.
[ ] layout BQ rerouté selon TI.
[ ] D+/D- USB-C vraie paire différentielles sur stack-up JLC réel.
[ ] ERC/DRC/BOM/PnP/polarités/testpoints revus.
[ ] PCB <=70 x 25 mm.
```

Les fonctions Android avancées du modem ne bloquent pas la PCB si les interfaces électriques sont prouvées.

---

# 15. Documents

```text
CDC courant        : CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md
MCU supervision V2 : MISSION_MCU_SUPERVISION_V2_2026-10-03.md
EC25 Android       : ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
J19                : MISSION_Q6A_NATIVE_BATTERY_BRINGUP_2026-09-25.md
STR historique     : CONCLUSIONS_STR_Q6A_2026-09-24_OTG_CONFIRME.md
```

Les documents du 30/09 et le CDC review du 02/10 restent utiles comme historique/audit, mais le présent état et le CDC du 03/10 ont priorité.