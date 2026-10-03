# Décisions consolidées — architecture Power / MCU / USB / EC25 — 2026-10-03

**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Rôle :** journal de décisions consolidé après revue profonde de l'architecture V0.5.  
**Référence électrique normative :** `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md`.

Ce document n'ajoute pas une architecture parallèle. Il résume les décisions qui ont été explicitement retenues afin d'éviter qu'un ancien document ou une ancienne hypothèse soit réintroduit pendant le schéma/PCB.

---

# 1. Architecture système retenue

```text
USB-C externe : 5 V + USB2 DEVICE uniquement
        |
        +--> BQ25628E --> SYS --> batterie 1S protégée
        |                  |
        |                  +--> MAIN_PWR  --> Q6A J19
        |                  +--> MODEM_PWR --> EC25 carrier BAT
        |                  +--> ANNEXE1
        |                  +--> ANNEXE2
        |                  +--> TPS610995 --> RP2040-Tiny
        |
        +--> D+/D- -----------------------> Q6A OTG/device
        +--> VBUS -- Schottky ------------> Q6A OTG VBUS

Q6A USB2 host #1 --> EC25 USB direct
Q6A USB2 host #2 --> caméra USB future
Q6A USB2 host #3 --> réserve / futur USB-A amovible
```

Décision : le port USB-C du téléphone reste volontairement simple. Il sert à la **charge, ADB, EDL, MTP/device**. Il ne fait pas host, DRP ou PD en V1.

---

# 2. USB-C externe

## 2.1 Rôle fixe device/sink

```text
CC1 -> 5.1 kOhm -> GND
CC2 -> 5.1 kOhm -> GND
```

Aucun contrôleur CC/DRP. Le Q6A n'expose pas une broche USB-C CC à déporter depuis son USB-A OTG ; son contrôleur USB sait changer de rôle logiciellement, mais ce mécanisme n'est pas utile puisque le port externe est désormais figé en device.

## 2.2 Mesure CC fusionnée

Une seule ligne CC est active selon l'orientation. Les deux mesures séparées GP26/GP27 sont supprimées.

```text
CC1 -- 470 kOhm --+
                   +--> CC_SENSE --> GP26 / ADC0
CC2 -- 470 kOhm --+
                   |
                 100 nF
                   |
                  GND
```

Le firmware distingue Default / 1.5 A / 3 A si la mesure est suffisamment nette ; tout cas ambigu retombe sur un courant conservateur.

## 2.3 VBUS vers Q6A

```text
USB-C VBUS -> B5819W SL / C8598 Basic -> Q6A OTG VBUS
```

La diode bloque un éventuel retour du 5 V Q6A vers le connecteur/BQ si le contrôleur était accidentellement forcé en host.

Gate physique : USB-C branché + MAIN_PWR OFF ne doit pas back-powerer significativement le Q6A via VBUS ou D+/D-.

---

# 3. Q6A : Power, suspend et heartbeat

Le réveil final n'utilise plus `GPIO59` comme ligne WAKE. Le MCU simule le **vrai bouton Power du Q6A**.

```text
GP8 -> 10 kOhm -> base S8050 C2146 Basic
base -> 100 kOhm -> GND
émetteur -> GND
collecteur -> Q6A PWR_ON_KEY
```

Cette ligne sert à :

```text
- cold boot après activation MAIN_PWR ;
- réveil depuis mem_sleep=deep ;
- événement bouton Power Android ;
- appui long de récupération si nécessaire.
```

La modification batterie native Q6A (`R7` retirée, `R24` montée) ne supprime pas cette fonction : il faut simplement que J19/MAIN_PWR alimente d'abord le PMIC avant de pulser `PWR_ON_KEY`.

## 3.1 SLEEP_REQ conservé

```text
GP9 -> Q6A GPIO58 = SLEEP_REQ
```

Le bouton Power n'est pas utilisé comme raccourci vers notre suspend final. `SLEEP_REQ` déclenche une procédure Q6A dédiée : préparation modem, USB host, OTG externe `none` si nécessaire, Wi-Fi/wake sources, puis vrai `mem_sleep=deep`.

## 3.2 GPIO59 devient HEARTBEAT

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

Le heartbeat est logiciel. Il peut coder RUN/transition et s'arrêter volontairement en deep. Son absence seule ne provoque jamais une coupure immédiate de MAIN_PWR.

---

# 4. MAIN_PWR et MODEM_PWR : OFF par défaut avec hold de reboot

Décision finale : les rails critiques sont **OFF par défaut**, mais une réinitialisation brève du RP2040 ne doit pas couper le Q6A ou le modem.

Le condensateur de temporisation est placé sur la grille du **petit NMOS de commande**, pas sur la grille du P-MOS de puissance.

Par rail :

```text
SYS -> source JMTQ55P02A
P-MOS drain -> charge
P-MOS gate -> 100 kOhm -> SYS
P-MOS gate -> drain AO3400A C20917 Basic
AO source -> GND

GPIO -> 10 kOhm -> gate AO3400A
gate AO -> 1 MOhm -> GND
gate AO -> 1 uF -> GND
```

Comportement cible :

```text
GPIO HIGH  : rail ON
GPIO LOW   : arrêt volontaire rapide
GPIO Hi-Z  : hold de l'ordre de 1 s
reset MCU  : rail reste ON le temps du reboot
MCU mort   : RC expire puis rail OFF
```

Le temps réel dépend du seuil du AO3400A et doit être mesuré. Cible pratique : environ 0.7 à 1.5 s.

Point de validation spécifique : lorsque le MCU est réellement **désalimenté** par `STORAGE_SW`, vérifier que `C_HOLD` ne back-powere pas le RP2040 de façon problématique à travers la résistance série/GPIO. Le Storage OFF doit être testé en plus du simple watchdog/reset MCU.

---

# 5. EC25 : alimentation principale séparée

Le VBUS USB du Q6A n'est plus l'alimentation principale LTE.

```text
SYS -> MODEM_PWR -> EC25 carrier BAT
Q6A USB2 host -> EC25 VBUS / DP / DN / GND
```

`BAT` fournit les pics de courant du modem. `VBUS` indique la présence du host USB et alimente le domaine USB.

Le câble USB Q6A <-> EC25 est **direct et hors carte Power**. D+/D- restent courts et torsadés, avec masse proche.

Aucun switch dédié `EC25_USB_VBUS` n'est ajouté en V1. Le Q6A doit suspendre correctement son port host pour permettre le sommeil modem. Si cela échoue, le fil VBUS du faisceau reste modifiable sans refaire immédiatement la carte Power.

Gate absolue : le `BAT` du carrier EC25 doit rester **strictement < 4.30 V**, y compris batterie pleine, USB branché et transitoires. Batterie 4.20 V classique uniquement ; pas de cellule HV 4.35/4.40 V.

Recovery modem retenu :

```text
AT propre -> PWRKEY -> MODEM_PWR power-cycle
```

---

# 6. EC25 RESET_N et GPIO de réserve

`RESET_N` n'est plus une fonction normale du MCU, car `MODEM_PWR` fournit déjà un dernier niveau de récupération.

`GP28` est conservé comme **GPIO de réserve**.

```text
GP28 -> TP_GP28_3V3
```

Prévoir en plus un étage optionnel DNP :

```text
GP28 -> 10 k DNP -> base S8050 DNP
base -> 100 k DNP -> GND
émetteur -> GND
collecteur -> TP_EC25_RST_OD -> strap DNP -> EC25 RESET_N
```

Ainsi GP28 reste utilisable directement en 3.3 V ; après l'étage optionnel, on dispose d'une sortie open-drain adaptée au domaine 1.8 V/RESET_N. Ce n'est pas une sortie push-pull 1.8 V.

Le `SN74AVC4T245` est déjà entièrement occupé par TXD/RXD/DTR/RI et ne fournit donc pas un cinquième canal libre.

---

# 7. SBS / batterie virtuelle Q6A

L'interface est conservée :

```text
GP14 <-> Q6A J20 pin 3 / GPIO24 / I2C6_SDA
GP15 <-> Q6A J20 pin 5 / GPIO25 / I2C6_SCL
```

Les pull-up ne doivent pas être alimentées par `3V3_MCU`, afin d'éviter le back-power du Q6A éteint.

Prévoir sur la carte Power :

```text
2 x 4.7 kOhm vers Q6A_3V3, DNP par défaut
```

Avant peuplement, mesurer si le Q6A fournit déjà les pull-up nécessaires. Si oui, les DNP restent non montées. Lorsque le Q6A est éteint, GP14/GP15 doivent rester en entrée/Hi-Z côté MCU tant que l'absence de back-power n'a pas été démontrée.

---

# 8. Pinout RP2040-Tiny consolidé

```text
GP0   -> EC25 RXD via SN74AVC4T245       (UART TX MCU)
GP1   <- EC25 TXD via SN74AVC4T245       (UART RX MCU)
GP2   -> EC25 DTR via SN74AVC4T245
GP3   <- EC25 RI via SN74AVC4T245
GP4   <-> BQ SDA
GP5   ->  BQ SCL
GP6   <-  BQ INT
GP7   <-  PWR_BUTTON utilisateur actif bas
GP8   ->  Q6A PWR_ON_KEY via S8050
GP9   ->  Q6A GPIO58 / SLEEP_REQ
GP10  ->  MAIN_PWR_EN + RC hold
GP11  ->  ANNEXE1_EN
GP12  ->  ANNEXE2_EN
GP13  ->  EC25 PWRKEY via S8050
GP14  <-> Q6A SBS_SDA
GP15  <-> Q6A SBS_SCL
GP26  <-  USB_CC_SENSE / ADC0 fusionné
GP27  <-  Q6A GPIO59 / HEARTBEAT
GP28  <-> RESERVE / TP 3.3 V / RESET_N OD DNP optionnel
GP29  ->  MODEM_PWR_EN + RC hold
```

Toutes les GPIO exposées ont une affectation, mais GP28 reste volontairement une réserve de dépannage.

---

# 9. Machine d'états attendue

```text
STORAGE : MCU/Q6A/EC25 éteints
OFF     : MCU ON ; MAIN/MODEM OFF
BOOT    : MAIN ON -> délai PMIC -> PWR_ON_KEY -> attendre heartbeat
RUN     : heartbeat normal
DEEP    : MAIN ON ; heartbeat peut être absent ; wake par PWR_ON_KEY
SHUTDOWN: Android/modem arrêtés proprement -> MODEM OFF -> MAIN OFF
FAULT   : tentative de recovery graduelle ; pas de hard cut sur simple absence heartbeat
```

Le bouton utilisateur est connecté au RP2040 uniquement. Le firmware décide entre cold boot, `SLEEP_REQ`, wake `PWR_ON_KEY` et recovery.

---

# 10. USB Q6A disponibles

Répartition retenue :

```text
USB2 HOST #1 -> EC25
USB2 HOST #2 -> caméra USB future
USB2 HOST #3 -> libre / futur connecteur USB-A amovible
USB3.1 OTG   -> USB-C externe en device uniquement
```

Cette répartition est précisément ce qui permet de supprimer la complexité DRP/CC-controller du port USB-C externe.

---

# 11. Points encore à prouver physiquement

L'architecture est figée, mais les points suivants restent des **gates de fabrication/bring-up**, pas des hypothèses à considérer comme déjà prouvées :

```text
[ ] Q6A J19 final : R7/R24/R190/R191/FB4 et 4.2/3.8/3.4/3.0 V.
[ ] PWR_ON_KEY : cold boot + wake deep avec Q6A alimenté par J19.
[ ] SLEEP_REQ/HEARTBEAT : séquence réelle et absence de faux fault en deep.
[ ] hold RC MAIN/MODEM : reset watchdog réel + Storage OFF + back-power GPIO.
[ ] EC25 BAT <4.30 V et stabilité pendant pics LTE combinés avec charge Q6A.
[ ] USB host EC25 réellement suspendu avec VBUS restant présent.
[ ] MODEM_PWR OFF + VBUS EC25 présent : pas de back-power significatif.
[ ] USB-C branché + MAIN_PWR OFF : pas de back-power Q6A.
[ ] SBS : présence/absence des pull-up Q6A mesurée avant peuplement ; GP14/15 Hi-Z Q6A OFF.
[ ] TPS610995 : shutdown/isolation et courant de stockage réels.
[ ] footprint RP2040-Tiny, FPC accessible, mécanique <=70 x 25 mm.
[ ] layout BQ refait selon TI et paire USB2 calculée sur le stack-up JLC réel.
```

---

# 12. Décisions explicitement abandonnées

```text
USB-C host / DRP / contrôleur CC
USB-PD V1
GPIO59 comme WAKE final
CC1 et CC2 sur deux ADC séparés
EC25 alimenté principalement par le VBUS USB du Q6A
USB EC25 routé dans la carte Power
switch VBUS EC25 dédié
EC25 RESET_N consommant une GPIO dédiée
latch complexe MAIN/MODEM
condensateur de 1 s directement sur la grille du P-MOS de puissance
codec audio Minimal EC25 / appels avec Q6A réellement OFF
```

Toute réintroduction d'un de ces choix doit être traitée comme une nouvelle révision d'architecture et non comme une petite correction du schéma.
