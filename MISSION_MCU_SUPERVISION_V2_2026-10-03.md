# MISSION — MCU superviseur V2 / Power + suspend + heartbeat Q6A

**Date :** 2026-10-03  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**CDC courant :** `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md`  
**État courant :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md`

Cette mission **supersède la mission V1 du 30/09 pour la carte finale**. La V1 reste un document historique de validation GPIO58/GPIO59. Dans la V2, `GPIO59` n'est plus le wake final : le réveil utilise le vrai `PWR_ON_KEY` du Q6A.

---

# 1. Objectif

Valider l'interface finale RP2040 <-> Q6A :

```text
1. cold boot du Q6A après activation MAIN_PWR ;
2. demande logicielle de suspend propre ;
3. vrai mem_sleep=deep ;
4. réveil par PWR_ON_KEY ;
5. heartbeat Q6A -> MCU ;
6. reset MCU sans coupure Q6A grâce au RC MAIN_PWR ;
7. shutdown propre avant coupure MAIN_PWR.
```

---

# 2. Câblage final

```text
RP2040 GP8  -> 10 k -> base S8050 -> Q6A PWR_ON_KEY
RP2040 GP9  -> Q6A J20 pin 37 / GPIO58 = SLEEP_REQ
Q6A J20 pin 36 / GPIO59 = HEARTBEAT -> RP2040 GP27
GND commun
```

SBS est traité séparément :

```text
GP14 <-> Q6A J20 pin 3 / I2C6 SDA
GP15 <-> Q6A J20 pin 5 / I2C6 SCL
```

---

# 3. PWR_ON_KEY

Driver :

```text
GP8 -> 10 kOhm -> base S8050 C2146
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Fonctions à prouver :

```text
[ ] Q6A alimenté J19 mais arrêté -> pulse PWRKEY -> cold boot
[ ] Q6A en deep -> pulse PWRKEY -> reprise Android
[ ] Q6A en RUN -> pulse court -> événement Power Android attendu
[ ] appui long simulé -> recovery/hard action documentée
```

Commencer avec des pulses de 200–300 ms et mesurer/ajuster.

---

# 4. SLEEP_REQ / GPIO58

But : ne pas utiliser l'événement Power Android comme raccourci vers le sleep final.

Le MCU déclenche `SLEEP_REQ`, puis un service/handler Q6A exécute la séquence :

```text
1. notifier la transition au MCU via heartbeat ;
2. préparer EC25 : QSCLK/DTR selon politique ;
3. demander le suspend USB du port host EC25 ;
4. mettre l'USB OTG externe en `none` si nécessaire ;
5. traiter Wi-Fi et autres wake sources ;
6. vérifier mem_sleep=deep ;
7. entrer en suspend.
```

Câblage recommandé :

```text
Q6A_3V3 -> 100 kOhm -> SLEEP_REQ
SLEEP_REQ -> 10 kOhm série -> GP9
```

Firmware RP :

```text
actif   : GP9 sortie LOW
repos   : GP9 input/Hi-Z
```

---

# 5. HEARTBEAT / GPIO59

Câblage :

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Le heartbeat est lent et logiciel. Proposition de sémantique initiale :

```text
LOW permanent au démarrage     : Q6A absent/non initialisé
~1 Hz                           : RUN normal
pattern rapide temporaire       : transition boot/shutdown/suspend
arrêt volontaire après annonce  : deep sleep
```

Les patterns exacts restent firmware ; seule la direction et le comportement électrique sont figés.

Règle de sûreté :

```text
heartbeat absent sans contexte connu
-> ne jamais couper MAIN_PWR immédiatement
-> tenter PWRKEY/wake
-> attendre timeout
-> collecter état
-> power-cycle seulement en dernier recours
```

---

# 6. MAIN_PWR hold pendant reset MCU

Hardware :

```text
GP10 -> 10 k -> gate AO3400A
AO gate -> 1 MOhm -> GND
AO gate -> 1 uF -> GND
AO drain -> gate JMTQ55P02A
P-gate -> 100 kOhm -> SYS
```

Test :

```text
Q6A RUN
-> reset RP2040
-> mesurer MAIN_PWR_OUT à l'oscilloscope
-> vérifier aucune chute suffisante pour reset Q6A
-> mesurer temps de hold réel
```

Firmware très tôt au boot :

```text
- lire niveau GP10 avant de le forcer ;
- si encore haut, le passer immédiatement output HIGH ;
- seulement ensuite initialiser le reste.
```

Même principe sur GP29/MODEM_PWR.

---

# 7. Machine d'états MCU

```text
STORAGE
  MCU OFF

OFF
  MCU ON
  MAIN OFF
  MODEM OFF

BOOT_Q6A
  MAIN ON
  délai PMIC
  pulse PWRKEY
  attendre heartbeat

RUN
  heartbeat vivant
  MAIN ON

PREP_SLEEP
  SLEEP_REQ
  attendre acquittement/pattern

DEEP
  MAIN ON
  heartbeat peut être absent
  réveil par PWRKEY

SHUTDOWN
  Android shutdown
  modem shutdown
  MODEM OFF
  MAIN OFF

FAULT
  recovery graduelle
```

Le MCU doit mémoriser le contexte `DEEP demandé` pour ne pas interpréter l'absence de heartbeat en deep comme un crash.

---

# 8. Bouton utilisateur

```text
GP7 <- bouton NO vers GND
pull-up 10 kOhm vers 3V3_MCU
```

Politique cible :

```text
Q6A OFF + appui       -> cold boot
Q6A RUN + appui court -> SLEEP_REQ
Q6A DEEP + appui      -> PWR_ON_KEY wake
appui long            -> menu/recovery selon firmware
```

Le bouton physique utilisateur n'est pas câblé directement au Q6A.

---

# 9. Critères de validation

```text
[ ] aucun pulse parasite PWRKEY/SLEEP_REQ au boot RP
[ ] cold boot Q6A 10/10
[ ] SLEEP_REQ reçu 20/20
[ ] vrai deep 10/10
[ ] wake PWR_ON_KEY 10/10
[ ] heartbeat RUN stable
[ ] deep reconnu sans faux FAULT
[ ] reset MCU en RUN sans reset Q6A
[ ] reset MCU avec MODEM_PWR actif sans reset modem
[ ] shutdown propre puis MAIN/MODEM OFF
[ ] back-power négligeable MCU OFF/Q6A OFF
```

---

# 10. Logs à archiver

```text
- traces RP2040 horodatées ;
- dmesg / logcat suspend-wake ;
- /sys/power/mem_sleep ;
- wake reason ;
- état USB host EC25 avant/après suspend ;
- état OTG externe ;
- captures oscilloscope MAIN_PWR/MODEM_PWR pendant reset MCU ;
- mesures heartbeat ;
- versions/SHAs des images et DTBO.
```

---

# 11. Point critique USB

Le STR déjà observé a montré que l'USB OTG externe actif peut empêcher le deep. La procédure finale doit donc explicitement gérer le contrôleur externe avant le suspend.

Le EC25 pose un second gate : son USB host doit entrer en suspend avec VBUS restant présent. Cette validation est nécessaire avant de considérer le mode deep finalisé.
