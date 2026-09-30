# PROJECT STATE — Maker Phone Q6A — état courant 2026-09-30

**Date :** 2026-09-30  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Statut :** **référence courante pour Power/MCU/modem/audio et préparation fabrication**

## Priorité documentaire

Pour les sujets **Power / batterie / MCU / modem / audio / Minimal Phone**, ce document et les documents spécialisés ci-dessous **supersèdent les choix plus anciens**, notamment les sections correspondantes de `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-22_MASTER_ANDROID_BOOT_OK.md`.

Ordre de référence courant :

```text
1. CDC_CARTE_POWER_MCU_V1_2026-09-30.md
2. PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-30_CURRENT.md
3. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
4. PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md
5. documents écran/STR spécialisés
6. anciens PROJECT_STATE / missions = historique ou contexte
```

Ne pas réintroduire automatiquement les anciennes architectures batterie 2S, codec Minimal EC25 ou MCU final différent sans nouvelle décision explicite.

---

# 1. Plateforme validée

```text
SBC            : Radxa Dragon Q6A V1.21
SoC            : Qualcomm QCS6490
Android        : Android 15 stock Radxa 20260630-b1, boot validé
Stockage       : eMMC YMTC 64 GB
Modem          : Quectel EC25-EUXGA
MCU V1         : Waveshare RP2040-Tiny
Batterie cible : Li-ion/LiPo 1S ~5000 mAh protégée
```

Android stock fonctionne sur le Q6A réel. Les futurs changements Android/modem doivent être intégrés par images ciblées et flash reproductible ; pendant la phase de développement, ADB/root/scripts manuels restent acceptés pour itérer rapidement.

---

# 2. Architecture Power/MCU courante

La référence électrique complète est `CDC_CARTE_POWER_MCU_V1_2026-09-30.md`.

Résumé :

```text
USB-C extérieur 5 V
    |
    +--> BQ25628E -> SYS -> batterie 1S / power-path
                         |
                         +--> MAIN_PWR -> Q6A J19
                         +--> ANNEXE1
                         +--> ANNEXE2
                         +--> TPS610995 -> 3.6 V -> RP2040-Tiny
                                              ^
                                              |
                                    STORAGE_SW sur EN

Q6A USB-A 2.0 host -> EC25 VBUS/D+/D-/GND
```

Décisions :

```text
- BQ25628E, charge max cible 2 A
- aucun USB-PD V1 ; entrée USB-C 5 V
- MAIN_PWR commandé par MCU
- EC25 alimenté par le VBUS USB-A host du Q6A
- aucun MODEM_PWR séparé
- deux sorties ANNEXE commutées
- PCB 4 couches
- dimensions maximales strictes 70 x 25 mm
- RP2040-Tiny soudé manuellement sur la face BOTTOM
```

---

# 3. Storage OFF MCU

Le téléphone doit pouvoir rester inutilisé plusieurs jours/semaines sans conserver la consommation ~1–2 mA du module RP2040-Tiny en sleep.

Architecture retenue :

```text
SYS -> VIN TPS610995
SYS -> pad STORAGE_SW_1
pad STORAGE_SW_2 -> EN TPS610995
EN -> pulldown 100 kOhm -> GND
```

Le switch mécanique est externe à la PCBA et soudé sur deux pads traversants accessibles.

Règle d'usage : **storage/hard-off uniquement après shutdown propre du Q6A**. Ce switch ne remplace pas la séquence normale d'extinction Android.

À valider avant fabrication : comportement shutdown du TPS610995 dans cette topologie. À mesurer après fabrication : courant résiduel et absence de back-power via les interfaces MCU.

---

# 4. Q6A batterie native J19

La cible est d'alimenter le Q6A comme système 1S via J19, l'électronique externe assurant charge/power-path.

Configuration matérielle cible du Q6A :

```text
R7    -> retiré / DNP
R24   -> 2 mOhm 1 % final
R190  -> 100 kOhm vers GND
R191  -> 10 kOhm vers GND
FB4   -> DNP
R185..R189 -> DNP si inspection physique confirme
```

Pour le premier bring-up, un **pont 0 Ohm / fil très court sur R24** est accepté temporairement. Il ne permet pas de valider correctement une mesure fuel-gauge basée sur le shunt final.

Premier test : alimentation de laboratoire ~3.8 V avec limitation de courant active, sans LiPo réelle.

Caractérisation avant fabrication souhaitée :

```text
4.2 V
3.8 V
3.4 V
3.0 V
```

Mesurer au minimum courant, puissance et température en idle, charge CPU et vrai STR.

---

# 5. Suspend / réveil MCU ↔ Q6A

Câblage retenu :

```text
Q6A pin 34 GND
Q6A pin 36 GPIO59 = MCU_WAKE
Q6A pin 37 GPIO58 = MCU_SLEEP_REQ
```

Objectif avant fabrication :

```text
GPIO58 -> demande logicielle de vrai mem_sleep=deep
GPIO59 -> réveil du Q6A depuis deep
```

Ce test est prioritaire car il valide directement le choix des GPIO de la carte Power/MCU.

---

# 6. EC25 — état courant

Modem réel :

```text
Quectel EC25-EUXGA
firmware EC25EUXGAR08A05M1G_01.001.01.001
USB VID:PID 2c7c:0125
```

Déjà prouvé :

```text
- commandes AT accessibles
- prototype RIL Quectel AIDL1 lancé sous Android
- 7 services Radio publiés
- Android atteint CARDSTATE_ABSENT / gsm.sim.state=ABSENT
```

Non encore prouvé :

```text
- SIM réelle détectée
- PIN
- enregistrement Orange
- appels
- SMS
- data QMI/RmNet
- audio appel
- IMS/VoLTE
```

Prochaine SIM de test : **Orange, PIN actif**. Une autre ligne est disponible pour appels/SMS.

Avant PCB, le minimum utile est : SIM détectée, PIN accepté, réseau et appel si possible, plus mesure `VIO/TXD/RI`. L'intégration complète Android peut continuer après commande.

---

# 7. EC25 ↔ MCU

La liaison principale Android ↔ modem reste USB depuis le Q6A.

La supervision MCU conserve :

```text
MCU UART TX/RX <-> EC25 RXD/TXD
MCU DTR        -> EC25 DTR
EC25 RI        -> MCU RI
MCU PWK/RST    -> drivers open-collector
```

Le level-shifter prévu est `SN74AVC4T245`, domaine EC25 alimenté par `VIO` attendu ~1.8 V et domaine MCU 3.3 V.

Mesurer **VIO, TXD, RI** sur le carrier réel avant connexion définitive.

L'UART MCU peut envoyer les commandes **AT** et recevoir les URC. `RI` reste le signal naturel d'événement asynchrone permettant au MCU de décider de réveiller le Q6A.

---

# 8. Minimal Phone — périmètre simplifié

L'ancien objectif « appel vocal complet avec Q6A physiquement OFF » est abandonné en V1 car il imposait une deuxième chaîne audio complète EC25 PCM + codec + mux/transducteurs.

Le Minimal Phone devient un **mode de supervision/interface basse consommation**, pas un deuxième téléphone complet.

Le MCU peut rester responsable de :

```text
- boutons physiques
- état power/batterie
- supervision EC25 par AT
- RI/DTR
- réveil du Q6A
- éventuellement mini-écran arrière basse consommation
```

Pour un appel entrant :

```text
EC25 -> RI -> MCU -> WAKE Q6A -> Android -> appel/audio
```

Donc tout appel vocal nécessite désormais le Q6A actif.

---

# 9. Architecture audio courante

L'ancienne architecture `EC25 PCM -> codec Minimal` est supprimée pour V1.

Le téléphone n'aura **pas de jack utilisateur**. Les écouteurs externes passent par Bluetooth.

Architecture retenue :

```text
Q6A codec / sortie HPH_L/R
        |
        +--> oreillette interne toujours raccordée
        |
        +--> entrée haute impédance ampli stéréo
                  +--> HP gauche
                  +--> HP droit

MCU -> alimentation/EN ampli stéréo via une sortie ANNEXE
micro interne -> entrée micro native Q6A à identifier/valider
```

Comportement :

```text
MEDIA : oreillette + 2 HP possibles
APPEL : ampli HP coupé par MCU -> oreillette seule
```

Il n'y a donc plus besoin de mux audio analogique pour la V1 si les tests électriques passent.

Tests nécessaires avant fabrication si possible :

```text
- HPH_L/R actif sans jack physique
- niveau/offset de sortie compatible avec l'oreillette
- entrée ampli suffisamment haute impédance pour ne pas charger HPH_L/R
- choix d'une entrée micro Q6A utilisable comme micro interne
```

---

# 10. Placement RP2040-Tiny / mécanique

Le module RP2040-Tiny est assemblé **face BOTTOM**, manuellement après PCBA.

Avant commande :

```text
- dessin officiel + mesure du module réel
- footprint SMT à castellations
- impression papier 1:1
- contrôle orientation
- congés de soudure accessibles
- keepout sous module
- aucune collision mécanique
- carte entière <= 70 x 25 mm
```

---

# 11. Philosophy de validation avant fabrication

Le projet ne cherche plus à prouver toutes les fonctions logicielles avant de commander la PCB. Si un problème non électrique apparaît ensuite, il sera traité par firmware, Android, câblage ou module externe.

Gates prioritaires avant commande :

```text
[ ] Q6A J19 inspection/rework minimal et boot sur alimentation labo
[ ] Q6A stable à plusieurs tensions 1S pertinentes
[ ] courant/puissance/température idle + CPU + STR
[ ] GPIO58 -> vrai deep
[ ] GPIO59 -> wake depuis deep
[ ] EC25 SIM Orange/PIN/réseau/appel si possible
[ ] EC25 VIO/TXD/RI mesurés
[ ] HPH_L/R audio sans jack physique
[ ] TPS610995 Storage OFF revu
[ ] analyse back-power MCU OFF
[ ] footprint RP2040 réel + 1:1
[ ] PCB <= 70 x 25 mm
[ ] ERC/DRC + BOM/footprints/polarités/testpoints/Gerbers revus
```

Non bloquants PCB : data Android complète, SMS complets, VoLTE, audio duplex final, démarrage autonome RIL, mesure exhaustive des transitoires batterie/LTE.

---

# 12. Documents spécialisés à utiliser

```text
Power/MCU final : CDC_CARTE_POWER_MCU_V1_2026-09-30.md
EC25/Android    : ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
MCU STR proto   : PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md
Mission STR MCU : MISSION_MCU_SUPERVISION_V1_2026-09-30.md
J19 native 1S   : MISSION_Q6A_NATIVE_BATTERY_BRINGUP_2026-09-25.md
```

Les anciens documents restent utiles pour l'historique et les preuves techniques, mais leurs architectures obsolètes ne doivent pas être recopiées dans le nouveau schéma sans vérifier ce document courant.
