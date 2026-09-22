# PROJECT STATE — Maker Phone / Radxa Dragon Q6A

**Date de gel :** 2026-09-16 11:13 CEST  
**Statut :** état de référence courant après audits matériel + Android + display + touch + EC25, avec **gel du périmètre de la PCB écran : carte minimale dédiée uniquement à l’écran et à son adaptation**.  
**Dernière mise à jour :** 2026-09-22 — inspection physique + bring-up initial terminés : Q6A réel `Dragon Q6A V1.21`, eMMC YMTC 64 GB, accès Qualcomm EDL/Sahara/Firehose validé, double sauvegarde QSPI usine vérifiée, firmware QSPI + Android 15 stock `20260630-b1` flashés avec succès, GPT eMMC post-flash validée et **premier boot Android réussi**. EC25 réel identifié `EC25-EUX / GA / EC25EUXGA-128-SGNS`; écran aftermarket `FMS780 JB2-1`. Le gel physique reste inchangé : **PCB minimale d’adaptation écran uniquement**.  
**Audience :** agent IA.  
**Règle :** ce fichier prime sur les anciens documents lorsqu'il indique qu'une décision a été remplacée.

---

# 0. Convention

Statuts :

```text
[CONFIRMÉ]      démontré par schéma / firmware / source / audit offline
[CIBLE]         architecture décidée pour la prochaine révision
[À VALIDER]     nécessite matériel, mesure ou choix final
[HISTORIQUE]    ancien choix conservé uniquement comme référence
```

Priorité des sources en cas de contradiction :

```text
1. CE DOCUMENT
2. rapports finaux touch/display + rapports finaux EC25/Quectel cités en section 22.7
3. Q6A_Android15_20260630_inspection_handoff_v10(4).md sections 100/101/102
4. schémas constructeur Radxa + Motorola + Quectel
5. SCHEMATIC_FROZEN_V1(5).md / V1 KiCad = ancienne baseline physique
6. fichiers MISSION_* = consignes de recherche, pas source de vérité finale
```

Ne pas réouvrir un point marqué `[CONFIRMÉ]` sans nouvelle preuve contradictoire.

---

# 1. But du projet

Maker Phone / plateforme de développement de poche autour du **Radxa Dragon Q6A / Qualcomm QCS6490**.

Objectifs :

```text
Android complet
écran smartphone Moto G200 direct MIPI-DSI
appels/SMS LTE via Quectel EC25-EUX (Cat 4)
mode "Minimal Phone" avec Q6A totalement éteint
batterie Li-ion 2S
MCU always-on
caméras
audio interne + jack
Maker Port protégé : GPIO / SPI / I²C / UART / PWM
fonctions labo possibles : ADC, UART terminal, I²C scanner, SPI console, logger
```

Le projet n'essaie pas d'égaler un smartphone industriel en finesse/photo/5G/étanchéité.

**Principe logiciel :** éviter un build AOSP complet. Kernel/DTBO/vendor ciblés seulement.

---

# 2. Plateforme Q6A

```text
Carte             Radxa Dragon Q6A
SoC               Qualcomm QCS6490 / famille BSP QCM6490 / YupikP
RAM               6 Go suffisants ; 12 Go préférables si budget
Stockage V1       Radxa eMMC Blue 64 GB
Dimensions carte  ~85 × 56 mm
```

Interfaces importantes :

```text
MIPI-DSI 4 lanes
CSI
USB 3.1 OTG / EDL
USB2 Host
Wi-Fi / Bluetooth
audio codec + J17/J18
header 40 pins
```

Noms de recherche BSP à considérer équivalents/convergents :

```text
QCS6490
QCM6490
YupikP
Lahaina
QCM6490.LA
```

---

# 3. Android / kernel Q6A audité

Image :

```text
Q6A-Android15-spi-emmc-boot-20260630-b1.7z
SHA-256 local vérifié : 1978e216beadb8bd9025bcab52814d1f0e16d01cbfee2fbd91c4795bd50ebb16
```

[CONFIRMÉ PAR CALCUL LOCAL UTILISATEUR] Cette empreinte devient la référence d’intégrité de l’archive stock utilisée pour les audits et futurs patchs. Toute image modifiée doit être distinguée explicitement de cette baseline.

Source Radxa :

```text
https://dl.radxa.com/dragon/q6a/images/android/Q6A-Android15-spi-emmc-boot-20260630-b1.7z
```

Kernel exact :

```text
5.4.295-qgki-debug-g646f05065a7e-dirty
```

Code Qualcomm identifié :

```text
tag    LA.UM.9.14.7.r1-02400-QCM6490.QISI15.0
commit 42e5df8082120c7dbfc646871caf1b361dbf4165
Clang indiqué par l'image : 11.0.2
```

Device Tree :

```text
vendor_boot.img
└── FDT #18 = base DTB exact Q6A

dtbo.img
└── entry 40 = overlay exact Radxa Dragon Q6A
```

DTBO :

```text
85 entrées
entry 40 stock = 148410 octets
round-trip DTC validé offline
```

Android est A/B. `dtbo` est une partition indépendante.

AVB, audit historique :

```text
vbmeta       : clé publique AOSP testkey RSA-4096 identifiée
vbmeta_system: clé publique AOSP testkey RSA-2048 identifiée
```

Toujours conserver backup/rollback avant flash.

## 3.1 Premier boot stock / qualification Android téléphone

[VALIDÉ SUR MATÉRIEL — 2026-09-22]

Le premier boot de l’image Radxa **strictement stock** a été réalisé avec succès sur le Q6A V1.21 équipé de l’eMMC YMTC 64 GB. Après flash du firmware QSPI associé puis de l’image Android eMMC officielle `20260630-b1`, l’utilisateur a atteint l’interface Android. La baseline « Q6A réel + eMMC réel + firmware stock + Android 15 stock » est donc désormais **BOOT VALIDÉ**.

La séquence de flash a été réalisée en EDL Qualcomm avec `edl-ng 1.6.0`. Le boot firmware QSPI d’origine a été sauvegardé deux fois avant toute écriture ; les deux dumps de 32 MiB sont bit-à-bit identiques (SHA-256 documenté en section 29).

Le fait que l’interface Android démarre ne prouve pas encore que l’image contient toute la pile « smartphone ». À inventorier explicitement dans `system`, `system_ext` et `product` / via ADB :

```text
com.android.phone
TelephonyProvider
CarrierConfig
Telecom
STK
features telephony
base APN
Dialer / Messaging
```

La **complétude de la pile téléphonie stock reste UNKNOWN** jusqu’à inspection via ADB (`pm list packages`, `pm list features`, services, APN, Telecom/Telephony). Le boot Android lui-même est désormais validé. La personnalisation visuelle téléphone (portrait, density/DPI, bars, overlays) est considérée secondaire par rapport à l’intégration modem/audio/power.

---

# 4. Architecture globale power / USB

## 4.1 Batterie

[CIBLE]

```text
2 × Li-ion/LiPo en série
~6,0 V déchargé
~7,4 V nominal
8,4 V chargé
```

Le Q6A accepte pratiquement **4,5–16 V** sur son entrée externe.

Donc :

```text
2S -> protections / switch -> entrée externe Q6A
```

**Pas de boost 2S -> 12 V pour le Q6A.**

Ne pas relier naïvement VBUS USB-C et batterie/Q6A en Y.

## 4.2 USB-C externe

[CIBLE]

```text
USB-C unique
├── data / OTG / EDL -> USB 3.1 Q6A via mux orientation SuperSpeed
└── VBUS / PD -> contrôleur PD -> chargeur 2S / power-path
```

Candidats actuels, non gelés :

```text
TPS25751D : contrôleur USB-C / PD
BQ25798   : chargeur buck-boost 1–4S + NVDC/power-path
BQ28Z610  : gauge/protection 1S/2S
```

## 4.3 Domaines

[CIBLE RÉVISÉE]

```text
2S
├── ALWAYS-ON / joignable
│   ├── MCU superviseur
│   ├── RTC
│   ├── fuel gauge / NTC
│   ├── boutons / watchdog
│   ├── mini écran basse consommation
│   ├── commande alimentation Q6A
│   └── EC25 core board : UART + RI + DTR ; PEN/PWRKEY selon validation
│
└── COMMUTÉ
    ├── Q6A
    ├── écran principal
    ├── caméras
    └── charges fortes
```

Le MCU n'est plus considéré comme un simple auxiliaire : il est le **superviseur matériel always-on** du téléphone.

Responsabilités cibles :

```text
- lecture de tous les boutons physiques
- démarrage du Q6A via son entrée PWR
- contrôle du switch/load-switch d'alimentation du Q6A
- extinction coopérative puis coupure physique du Q6A
- watchdog / récupération en cas de blocage
- contrôle du mode Minimal Phone
- contrôle de l'EC25 par UART lorsque le Q6A est éteint
- surveillance de RI et contrôle de DTR pour veille/réveil modem
- gestion du mini-display et de l'interface Minimal
- communication bidirectionnelle avec le Q6A
```

MCU :

```text
Prototype : Raspberry Pi Pico / RP2040 accepté et préféré pour la facilité de développement
Final     : MCU low-power exact non gelé ; STM32U5 reste une famille candidate
```

Le choix d'un MCU final différent du RP2040 serait motivé principalement par la consommation always-on et l'intégration, pas par un manque de performances du Pico.

États visés :

```text
Performance
écran éteint / DVFS
Suspend-to-RAM : Q6A suspendu + EC25 Sleep + MCU actif
Minimal Phone  : Q6A OFF, MCU + EC25 enregistré/Sleep + mini-display + audio Minimal disponibles
Deep/storage   : Q6A + EC25 OFF, MCU/RTC/gauge seulement
```

[CONFIRMÉ AU NIVEAU MODULE] Le EC25 dispose d'un **Sleep Mode** qui permet de rester enregistré/joignable tout en réduisant fortement sa consommation. La cible de veille est donc de laisser le modem alimenté et enregistré, avec `AT+QSCLK=1`, `DTR` dans l'état autorisant le sommeil et `RI` surveillé par le MCU. Un appel/SMS/URC peut alors provoquer `RI`, puis le MCU réveille le Q6A si Android doit reprendre la main.

[À VALIDER SUR LA CORE BOARD] Lorsque le Q6A est complètement OFF, il faut confirmer que le lien USB ne crée ni back-power ni réveil permanent du EC25. Le contrôle de `USB_VBUS` ou un switch USB/power dédié peut devenir nécessaire.

Mini-display : Memory LCD actuellement préféré, référence non choisie.

## 4.4 Communication MCU <-> Q6A et boutons

[CIBLE]

Tous les boutons physiques du téléphone arrivent **une seule fois sur le MCU**. Ils ne sont pas doublés électriquement vers le Q6A.

Le bouton Power reste une fonction matérielle spéciale :

```text
bouton physique -> MCU
MCU -> transistor/open-drain -> entrée PWR Q6A
MCU -> commande du switch d'alimentation principal Q6A
```

Le MCU doit pouvoir allumer le Q6A alors que celui-ci est totalement hors tension.

Pour les boutons Android ordinaires (Volume + / Volume - / fonctions), la cible est :

```text
bouton -> MCU
MCU -> événement dans FIFO
MCU_IRQ -> GPIO Q6A
Q6A -> lecture I²C de la FIFO
driver kernel -> Linux input EV_KEY
Android -> KEYCODE normal
```

Le Q6A est maître I²C ; le MCU est esclave I²C et dispose d'une ligne d'interruption `MCU_IRQ` pour signaler les événements asynchrones.

Ne pas implémenter les boutons principaux par une application Android qui scrute périodiquement I²C. Le chemin normal doit être un petit driver kernel/input afin que les événements soient natifs pour Linux/Android.

Le bus I²C Q6A exact reste [À VALIDER] après audit pinmux. **Ne pas réutiliser SPI6/QUP0_SE6 réservé au tactile** sur les broches header déjà affectées.

Réserver si possible deux GPIO/pads de secours entre MCU et Q6A pour Volume +/- matériel direct. Ils ne sont pas nécessaires au fonctionnement Android normal mais gardent une voie de repli pour bootloader/recovery si le firmware de pré-boot ne sait pas lire le MCU I²C.

Extinction normale :

```text
1. demande utilisateur / MCU
2. demande de shutdown au Q6A
3. Android/Linux effectue sync + arrêt propre
4. Q6A signale SAFE_TO_CUT au MCU
5. MCU coupe physiquement le rail Q6A
```

Le mécanisme exact `SAFE_TO_CUT` peut être un GPIO dédié ou un notifier/driver kernel associé à la liaison MCU ; il reste [À VALIDER] dans le kernel Qualcomm 5.4 exact.

Extinction forcée : autorisée uniquement après appui long/timeout lorsque le Q6A ne répond plus.

La communication I²C générale MCU/Q6A pourra aussi transporter :

```text
version MCU
état power
cause de wake
état EC25
batterie / températures
diagnostic
demandes de mode
événements non critiques
```

---

# 5. Écran Moto G200 / Edge S30

## 5.1 Module cible

```text
Moto G200 5G / Edge S30
codename xpeng
6,8"
1080 × 2460
module no-frame
référence OEM attendue : Tianma NT36672E
```

Le module réellement acheté est aftermarket : identité/révision/144 Hz à confirmer physiquement.

Connecteur côté écran :

```text
Panasonic AXE540127
40 contacts
pitch 0,4 mm
LCSC C3652918
mating réel à confirmer sur pièce
```

## 5.2 Interface display confirmée

```text
MIPI DSI
D-PHY
4 lanes + clock
RGB888
video mode
burst mode
DSC
```

Modes Moto documentés :

```text
144 / 120 / 90 / 60 / 50 / 48 Hz
```

Bring-up obligatoire :

```text
60 Hz d'abord
puis 90 -> 120 -> 144
DFPS/QSync en dernier
```

DSC Moto :

```text
slice height      20
slice width       540
slices/packet     2
bpc               8
bpp               8
block prediction  enabled
```

Status DCS utile :

```text
read 0x0A -> attendu 0x9C
```

Reset Moto retenu :

```text
1 / 5 ms
0 / 1 ms
1 / 10 ms
```

Séquence DCS complète Moto/Tianma obligatoire ; ne pas réduire à seulement 0x11/0x29.

---

## 5.3 Alternative étudiée — écran Redmi Note 7

[ÉTUDE COMPARATIVE / **NON ADOPTÉE**]

Le Redmi Note 7 a été examiné comme alternative potentiellement plus simple au Moto G200. Certaines variantes publiques sont documentées comme dalles 1080×2340 MIPI-DSI 4 lanes à fonctionnement plus conventionnel et sans exigence DSC comparable à notre bring-up Moto. Cela pourrait simplifier le premier affichage 60 Hz.

Mais « écran Redmi Note 7 » n’est pas une référence unique : plusieurs fournisseurs/contrôleurs de dalle existent, et l’identité exacte du tactile/bus de chaque assemblage aftermarket n’est pas suffisamment figée pour justifier une migration.

Décision courante :

```text
Moto G200 / Edge S30 reste la cible écran V1
Redmi Note 7 = alternative documentaire seulement
aucun redesign autour du Redmi Note 7 sans référence dalle + FPC + tactile exacts
```

Cette étude ne remplace aucune décision des sections 5 à 12.

---

# 6. Conclusion logicielle DISPLAY

[CONFIRMÉ]

**Classification : A — DTBO-only. Aucun patch kernel display requis.**

Le Q6A contient déjà :

```text
qcom,dsi-display-primary
mdss_dsi0
mdss_dsi_phy0
NT36672E D-PHY / DSC
profils 144 / 120 / 90 / 60
parser Qualcomm :
  DCS ON/OFF
  timings
  DSC
  reset
  TE
  status check
  DFPS
```

Le round-trip entry 40 et un overlay structurel de sélection NT36672E ont compilé offline.

Premier bring-up prévu :

```text
1080×2460
D-PHY 4 lanes
video burst
DSC
politique 60 Hz
backlight OFF
status 0x0A == 0x9C avant backlight
```

Restent matériels :

```text
GPIO/pinctrl RESET panneau Q6A
GPIO/pinctrl TE Q6A
rails VSP/VSN externes réellement commandés
intégrité DSI du PCB
identité écran aftermarket
```

---

# 7. Pinout utile FPC Moto

Pins utiles confirmés :

| Pin | Fonction |
|---:|---|
| 1 | IOVCC / VDDIO 1,8 V |
| 3 | VSP +5,5 V |
| 4 | DISP_RST_N |
| 5 | VSN ~-5,7 V |
| 6 | DISP_TE |
| 8 | DISP_PWM_OUT — sortie panneau ; ne pas utiliser comme entrée BL Q6A |
| 11 | WLED_A anode commune |
| 12/14 | DSI DATA2+ / DATA2- |
| 13 | WLED_K1 |
| 15 | WLED_K2 |
| 17 | WLED_K3 |
| 18/20 | DSI DATA1+ / DATA1- |
| 21 | TP_INT_N |
| 24/26 | DSI CLK+ / CLK- |
| 27 | TP_RST_N |
| 30/32 | DSI DATA0+ / DATA0- |
| 31 | TP_SPI_CS_N |
| 33 | TP_SPI_MISO |
| 35 | TP_SPI_MOSI |
| 36/38 | DSI DATA3+ / DATA3- |
| 37 | TP_SPI_CLK |

GND : 10, 16, 22, 28, 29, 34, 39, 40.

---

# 8. Tactile Moto — faits établis

```text
bus                 SPI
niveau côté tactile 1,8 V
binding Moto        "novatek,NVT-ts-spi"
famille             Novatek NT36xxx
trim supporté       NT36675
architecture        0-flash / host-download firmware
spi-max-frequency   9,6 MHz dans xpeng
bring-up conseillé  4,8 MHz
```

Firmware runtime exact :

```text
novatek_ts-NT36675-21101302-6044-xpeng.bin
taille     139264 octets
SHA-256    f4badcc32fd19fd3b81bb2596418a0c4371c83ede1cbe3dc9b1819c9fdb0676f
```

Firmware MP/test, ne pas utiliser en runtime :

```text
mp_novatek_ts-NT36675-21101302-6044-xpeng.bin
```

Le script xpeng/Lineage sélectionne explicitement le firmware normal pour `panel_supplier=tianma`.

---

# 9. Conclusion logicielle TOUCH

[CONFIRMÉ]

Le driver Novatek stock Q6A :

```text
CONFIG_TOUCHSCREEN_NT36XXX=y
binding "novatek,NVT-ts"
i2c_driver
i2c_add_driver()
```

Il est **incompatible structurellement** avec le tactile Moto SPI.

Driver de référence à reprendre :

```text
LineageOS/android_kernel_motorola_sm7325
drivers/input/touchscreen/nova_0flash_mmi/
```

Cœur minimal :

```text
nt36xxx.c
nt36xxx.h
nt36xxx_fw_update.c
```

À exclure au premier bring-up :

```text
Motorola MMI
MP/factory
procfs étendu
stylus
gestures avancées
edge suppression
extensions panel Motorola
```

Ne pas activer `CONFIG_SPI_SM8450`.

## Stratégie finale

**Classification D1 : rebuild contrôlé du kernel Q6A avec driver Novatek SPI minimal intégré (`=y`).**

Pourquoi pas `.ko` externe :

```text
CONFIG_MODVERSIONS=y
archive stock sans Module.symvers exact
pas d'include/generated exact
ABI module non reproductible proprement
```

Un module externe reste plan B seulement si Radxa fournit les artefacts ABI du build exact.

**Cela ne signifie PAS recompiler tout Android.**
Rebuild ciblé kernel + DTBO + firmware/vendor + packaging/AVB.

---

# 10. SPI tactile Q6A

[CIBLE / mapping désormais retenu]

Header Q6A, logique **3,3 V** :

| Signal | Header Q6A | GPIO | Fonction |
|---|---:|---:|---|
| TP_SPI_MISO | 3 | GPIO24 | SPI6_MISO |
| TP_SPI_MOSI | 5 | GPIO25 | SPI6_MOSI |
| TP_SPI_CLK | 16 | GPIO26 | SPI6_SCLK |
| TP_SPI_CS_N | 18 | GPIO27 | SPI6_CS0 |

Contrôleur :

```text
QUP0 / SE6 / SPI6
```

RESET/IRQ proposés :

```text
TP_RST_N -> header 7  / GPIO96
TP_INT_N -> header 12 / GPIO97
```

Avant fabrication : vérifier une dernière fois pinmux/conflits GPIO96/97 dans le DT final.

Important :

```text
Q6A header SPI = 3,3 V
Moto tactile   = 1,8 V
=> level shifting obligatoire
```

Le bloc touch J10 Q6A d'origine ne suffit pas : il expose après UM3204 Reset/INT/SDA/SCL mais pas le CLK+CS SPI complet requis par xpeng.

---

## 10.1 Alternative tactile — RP2040 / USB HID Digitizer

[PROPOSITION / À PROTOTYPER — **NON FIGÉE, NE REMPLACE PAS LA STRATÉGIE SPI DIRECTE ACTUELLE**]

Clarification obligatoire : le tactile Moto retenu n’est **pas I²C**. Il reste un **Novatek NT36675 sur SPI 1,8 V**, architecture 0-flash avec téléchargement du firmware par l’hôte.

Alternative étudiée : déplacer la logique spécifique Novatek hors du kernel Q6A et la porter sur un RP2040/Pico :

```text
NT36675 tactile Moto
        │ SPI 1,8 V + IRQ/RESET
        ↓
4× 74LVC2T45 / adaptation 1,8↔3,3 V
        ↓
RP2040 / Pico
  - reset + trim ID
  - téléchargement firmware 139264 octets
  - lecture/décodage paquets tactiles
        ↓
USB Device HID Digitizer / multitouch standard
        ↓
Q6A USB Host -> hid-multitouch -> Android InputReader
```

Intérêt potentiel :

```text
+ supprimer le driver Novatek spécifique du kernel Q6A
+ découpler le tactile du BSP Qualcomm
+ debug firmware MCU plus simple
+ Android reçoit un périphérique tactile HID standard
```

Contraintes / risques :

```text
- un USB Host interne consommé ; Host #3 est actuellement réserve
- 74LVC2T45 restent nécessaires côté tactile 1,8 V
- descriptor HID multitouch correct obligatoire ; éviter Raw HID propriétaire
- suspend / USB remote-wakeup / double-tap-to-wake à valider
- latence, fréquence SPI et stabilité 10 doigts à mesurer
```

Décision de méthode : **faire un prototype Pico externe avant toute modification de la PCB écran**. Si Android reconnaît proprement le périphérique comme écran tactile interne et que le suspend/wake est acceptable, cette architecture pourra être reconsidérée.

---

# 11. Interposer écran — CIRCUIT HISTORIQUE ACTUEL EN KICAD

[HISTORIQUE / référence V1, ne pas considérer comme architecture finale]

`SCHEMATIC_FROZEN_V1(5).md` + `V1 (2)(1).zip` décrivent :

```text
DSI Q6A J10 -> 3× TPD4E05U06 -> écran Moto
VDD 1,8 V direct
TPS65131 pour +5,5/-5,7 V
AO3401A disconnect VSP
TPS61194 depuis batterie 2S pour backlight
4× 74LVC2T45 pour tactile/contrôles 1,8↔3,3
GPIO auxiliaires via pads de bring-up
```

Ancienne BOM critique :

```text
J écran   AXE540127                 C3652918
U bias    TPS65131RGER              C87663
U BL      TPS61194PWPR              C2678565
U level   74LVC2T45DC,125 ×4       C730237
U DSI ESD TPD4E05U06DQAR ×3        C138714
Q PMOS    AO3401A                   C15127
```

Le PCB de cette archive n'était pas réellement routé/finalisé ; conserver comme référence schématique/BOM seulement.

---

# 12. Interposer écran — PCB MINIMALE, PÉRIMÈTRE FIGÉ

Le projet a explicitement décidé de **redessiner** l’interposer et de supprimer les blocs devenus inutiles.

## 12.-1 Gel de périmètre physique — décision du 2026-09-16

[CIBLE FIGÉE]

La carte à concevoir maintenant est une **PCB minimale dédiée uniquement à l’écran Moto G200/xpeng et à son adaptation vers le Q6A**. Ce gel concerne le **périmètre physique de cette PCB**.

Fonctions autorisées sur cette carte :

```text
- connecteur Q6A display / interconnexion vers J10 selon le montage final
- connecteur écran Moto 40 pins
- routage MIPI-DSI nécessaire au panneau
- IOVCC 1,8 V nécessaire au module écran/tactile
- adaptation tactile SPI/GPIO 3,3 V <-> 1,8 V
- génération des rails LCD VSP/VSN
- adaptation du backlight Moto au circuit backlight Q6A retenu
- passifs, protections strictement nécessaires, straps et points de test liés à l’écran
- pads de programmation nécessaires au TPS65132W
```

Sont **hors périmètre de cette PCB** jusqu’à nouvelle décision explicite :

```text
- MCU superviseur / RTC / watchdog / boutons téléphone
- Maker Port et ses protections/instruments
- USB-C externe, USB-PD, chargeur 2S, fuel gauge, protections batterie
- switch/load-switch principal d’alimentation Q6A
- modem EC25 et logique de contrôle modem non requise par l’écran
- codec/audio Minimal, ampli haut-parleur et mux audio
- caméras, capteurs généraux, haptique
- distribution de puissance générale du téléphone hors rails consommés par l’écran
- toute autre fonction ajoutée uniquement pour économiser une future PCB
```

Les réflexions de consolidation mécanique/électronique menées autour d’une éventuelle « Main Board » sont **mises en suspens**. Elles ne constituent pas une cible de fabrication. La décision actuelle est volontairement conservatrice : **une carte écran = écran + adaptation écran uniquement**.

Cette décision n’annule pas les architectures fonctionnelles MCU, power, modem, audio ou Maker Port décrites ailleurs dans ce document ; elle impose seulement qu’elles soient traitées séparément lors du futur découpage du téléphone.

## 12.0 Résumé électrique de la carte cible

[CIBLE FIGÉE]

```text
2 connecteurs : Q6A J10 <-> Moto 40 pins
DSI : direct, sans TPD4E05U06 sur l’interposer
IOVCC écran/tactile : 1,8 V Q6A direct
Touch SPI / GPIO : 4× 74LVC2T45, directions fixes, 3,3 V <-> 1,8 V
Bias LCD : TPS65132WRVCT (WQFN-20), +5,5 V / -5,7 V après programmation NVM
Programmation bias : 4 pads SMD 2,54 mm, sans trous
Backlight : SY7203 natif Q6A + adaptation passive des 3 cathodes Moto
Ballast : 47 Ω sur K1/K2/K3
Mesure ballast : 2 points de test par résistance, donc 6 au total
Courant BL : RSET sélectionnable par straps 0 Ω ; premier essai ~8,2 Ω
```

Blocs **supprimés de la cible** :

```text
TPS61194
TPS65131
AO3401A de l’ancien bias
3× TPD4E05U06 sur DSI
```

Le PCB final n’est pas encore routé ; les choix ci-dessus sont le point de départ obligatoire du nouveau schéma. **Le schéma doit rester strictement dans le périmètre écran/adaptation défini en 12.-1.**

## 12.1 DSI

Ancien :

```text
Q6A -> 3× TPD4E05U06 -> Moto
```

Cible figée :

```text
Q6A -> Moto directement
```

Raison : interposer interne court/permanent, réduction de capacité/parasitages et absence de TVS DSI discrètes équivalentes dans les chemins OEM observés.

Décision : **supprimer les 3 TPD4E05U06 du chemin DSI**.

Précautions prototype : pas de hot-plug, manipulation ESD propre, impédance différentielle et retour de masse soignés.

## 12.2 Bias LCD — TPS65132WRVCT

[CIBLE FIGÉE]

Le `TPS65131RGER` historique est abandonné. Motorola utilise le **TPS65132A0YFFR** sur le G200/xpeng, mais cette version DSBGA n’est plus la cible de l’interposer car elle peut imposer le service JLCPCB Standard. La cible devient le **TI TPS65132WRVCT**, en **WQFN-20 3 × 4 mm**, afin de conserver la même fonction tout en restant compatible avec **JLCPCB Economic PCBA**.

Rails Moto :

```text
VSP = +5,5 V
VSN = -5,7 V
```

L’asymétrie est volontaire : le schéma Moto indique explicitement +5,5 V et -5,7 V. Ne pas remplacer par ±5,5 V uniquement pour simplifier.

Architecture :

```text
Q6A 5 V -> VIN TPS65132W
1 seule inductance
OUTP -> VSP +5,5 V
OUTN -> VSN -5,7 V
ENP / ENN -> commande bias normale [implémentation GPIO à finaliser]
```

Réseau de puissance retenu, basé sur le schéma Motorola et l’application TI :

```text
L      = 2,2 µH
CIN    = 4,7 µF
CREG   = 10 µF
COUT+  = 10 µF
COUT-  = 10 µF
CFLY   = 4,7 µF
```

Des céramiques 25 V peuvent être utilisées pour garder de la marge DC-bias ; les références Basic 4,7 µF / 25 V et 10 µF / 25 V existent dans le snapshot JLC. L’inductance doit être une vraie inductance de puissance 2,2 µH avec courant admissible de l’ordre de la référence TI/Moto ; **ne pas utiliser les petites inductances Basic 0603/0805 du snapshot**, qui ne conviennent pas en courant.

Le composant exact d’inductance/footprint reste [À VALIDER] au moment du schéma/BOM selon stock.

### Programmation NVM du bias

Le `TPS65132WRVCT` permet de programmer séparément les deux rails par I²C par pas de 100 mV et de mémoriser la configuration de façon non volatile. La variante W démarre autour de ±5,4 V par défaut ; la cible du projet reste **+5,5 V / -5,7 V**.

Cible de programmation :

```text
VPOS = +5,5 V
VNEG = -5,7 V
sauvegarde non volatile
```

**Les octets exacts de registres et la séquence d’écriture NVM doivent être revérifiés dans la datasheet du TPS65132W avant de figer le script de production.** Ne pas reprendre aveuglément les valeurs précédemment notées pour la variante A0.

Le logiciel de programmation doit **relire puis faire un power-cycle et revérifier** avant de déclarer PASS. Ne pas brancher la dalle comme produit fini tant que la programmation n’a pas été vérifiée.

Programmateur retenu :

```text
PC -> USB -> Raspberry Pi Pico -> petit adaptateur/jig -> interposer
```

Le PC lance un petit programme Python ; le Pico exécute les transactions I²C et renvoie PASS/FAIL. Le Pico/jig porte l’éventuelle adaptation de niveau et les pull-ups de programmation ; ne pas encombrer le téléphone avec un connecteur de programmation.

Interface physique figée sur l’interposer :

```text
4 pads SMD nus, sans trous, pas 2,54 mm
ordre : GND | SDA | SCL | +5V
zone accessible au bord du PCB
pads assez grands pour être touchés avec des pins/header standards
aucun header monté
```

Le bus I²C du TPS65132 accepte un niveau haut dès ~1,1 V ; le jig peut donc travailler côté Pico en 3,3 V avec une adaptation appropriée si des pull-ups 5 V sont utilisés. La réalisation exacte du petit jig n’est pas une charge permanente du PCB téléphone.

### Pourquoi TPS65132WRVCT

```text
TPS65131      : exact électriquement mais coûteux, 2 inductances, diodes, feedbacks, compensation, grosse BOM
TPS65132B5    : très simple mais fixe +5,5/-5,5 V, donc ne reproduit pas le rail -5,7 V Moto
TPS65132A0YFFR: référence OEM Moto, mais DSBGA fin pouvant imposer JLCPCB Standard
TPS65132WRVCT : même famille/fonction, 1 inductance, rails programmables, WQFN-20, compatible Economic PCBA
```

**Décision figée : l’interposer doit rester en JLCPCB Economic PCBA.** Le TPS65132A0YFFR reste une référence OEM utile, mais le composant à placer sur notre carte est le **TPS65132WRVCT**. Les prix et stocks JLC/LCSC restent à revérifier au moment de commander.

## 12.3 Backlight — SY7203 natif Q6A

[CIBLE FIGÉE POUR PROTOTYPE / À VALIDER SUR MATÉRIEL]

Le `TPS61194` est supprimé de l’interposer. Le prototype réutilise le boost backlight natif Q6A `U43 = SY7203DBC`.

Q6A natif :

```text
VCC_5V_PERI -> L48 4,7 µH -> D14 B5819W -> VCC_LEDA
U43 SY7203DBC
EN/PWM = EDP_BLPWM
FB = VCC_LEDK
R315 stock = 1,1 Ω
J10 LED- = pins 34/35
J10 LED+ = pins 38/39
C644 schéma = 2,2 µF / 10 V sur VCC_LEDA
```

Le SY7203 utilise une référence FB d’environ 0,2 V :

```text
I_LED(total) ≈ 0,2 / RSET
```

Le `R315 = 1,1 Ω` du Q6A correspond à ~182 mA, ce qui est cohérent avec l’écran officiel Radxa Display 10 FHD (~180 mA, 12–14 V). Il **n’est donc pas une erreur de conception** ; il est simplement trop agressif pour notre premier essai sur les trois strings Moto.

Avant tout essai Moto :

```text
1. retirer/désactiver R315 1,1 Ω
2. vérifier physiquement C644 [À VALIDER SUR MATÉRIEL]
3. si C644 réellement monté est seulement 10 V : le retirer et le remplacer/bodger par 2,2 µF / 50 V X7R près de U43/D14
4. si C644 monté est déjà une référence >= 50 V : ne pas le remplacer inutilement
```

Le marquage `10 V` du PDF Q6A est incohérent avec l’écran officiel 12–14 V et avec plusieurs autres cartes Radxa utilisant le même SY7203 et un 2,2 µF / 50 V. Ne pas inventer un clamp 10 V.

### Connexion Moto

```text
Q6A LED+ / VCC_LEDA -> Moto WLED_A

Moto K1 -> TP_K1A -> 47 Ω -> TP_K1B ->+
Moto K2 -> TP_K2A -> 47 Ω -> TP_K2B ->+-> Q6A LED- / VCC_LEDK / FB
Moto K3 -> TP_K3A -> 47 Ω -> TP_K3B ->+
                                             |
                                             -> RSET sélectionnable -> GND
```

Chaque ballast 47 Ω a **deux petits points de test**, placés directement à ses deux extrémités : 6 points de test au total. Ils servent à mesurer la chute différentielle sans se référencer à la masse.

Calcul :

```text
I_Kx = ΔV_ballast / 47 Ω
```

Exemples :

```text
0,47 V  -> 10 mA
0,705 V -> 15 mA
0,94 V  -> 20 mA
1,175 V -> 25 mA
```

Les TP ballast peuvent être de petits pads SMD nus (~1 à 1,5 mm typ.), accessibles aux pointes de multimètre. Ils ne doivent pas être confondus avec les gros pads 2,54 mm de programmation bias.

### RSET sélectionnable

Le réseau RSET doit permettre plusieurs valeurs via résistances candidates + straps `0 Ω`.

Règle obligatoire : **une seule branche de sélection doit être fermée à la fois**, sauf calcul volontaire explicite. Deux straps simultanés mettraient les résistances en parallèle et augmenteraient le courant.

Premier essai figé :

```text
RSET ≈ 8,2 ΩI_total ≈ 0,2 / 8,2 ≈ 24,4 mA
```

C’est volontairement conservateur : même si une seule des trois strings prenait presque tout le courant, elle resterait proche du courant OEM d’une string (~24,8 mA).

Après mesure des trois chutes sur les 47 Ω, on pourra réduire RSET progressivement si le partage est satisfaisant. Les valeurs plus agressives ne sont **pas encore figées**.

Risque principal : le SY7203 régule le **courant total**, pas trois sinks indépendants. Une string au Vf plus faible peut prendre plus de courant. Arrêter l’essai si le partage est mauvais, si une branche approche un courant excessif, ou si VCC_LEDA se rapproche de l’OVP ~30 V.

## 12.4 Level shifting tactile

[CIBLE FIGÉE]

On conserve la solution historique **74LVC2T45 directionnelle** plutôt qu’un auto-directionnel TXS/TXB.

Baseline PCB :

```text
4× 74LVC2T45
domaine A = 1,8 V écran/tactile
domaine B = 3,3 V Q6A/J20
```

Directions typiques :

```text
3,3 -> 1,8 : SPI CLK, MOSI, CS, RESET / contrôles concernés
1,8 -> 3,3 : SPI MISO, IRQ / retours concernés
```

Le tactile doit démarrer à 4,8 MHz puis viser 9,6 MHz. Les `74LVC2T45` sont suffisamment rapides pour cette plage et offrent une direction déterministe.

Ne pas utiliser `2N7002/BSS138` comme solution générique pour le SPI push-pull MHz.

Attention : ne pas traduire automatiquement RESET/TE display si la source réelle est déjà le connecteur display Q6A en 1,8 V. L’affectation finale de chaque canal doit suivre le pinout réel choisi au schéma.

---

# 13. Backlight Moto — référence OEM et justification

Circuit OEM Moto :

```text
AW99703CSR
3 cathodes indépendantes
anode commune
OVP ~31 V
~24,8 mA/string
PWM
```

Pins FPC :

```text
WLED_A = 11
K1     = 13
K2     = 15
K3     = 17
```

La cible SY7203 ne reproduit pas les trois sinks indépendants. Les trois `47 Ω` servent uniquement de ballast/amélioration de partage et de résistances de mesure ; ils ne transforment pas le SY7203 en driver multicanal.

Le driver multicanal dédié reste un **fallback** si les mesures de partage sont mauvaises. Il n’est pas monté dans la première révision de test.

Le grand écran officiel Radxa n’est pas une preuve que les trois cathodes Moto peuvent être mises en parallèle sans risque : l’écran Radxa présente une interface backlight globale dont le matching interne n’est pas documenté publiquement, alors que Moto expose explicitement trois cathodes.

---

# 14. JLCPCB / stratégie composants

Contrainte figée : **la carte doit rester compatible JLCPCB Economic PCBA**. Choisir des composants compatibles Economic et utiliser des références **JLCPCB Basic** partout où cela ne dégrade pas l’architecture. Un composant qui forcerait le passage global en Standard doit être remplacé si une solution fonctionnellement équivalente et raisonnable existe.

Snapshot :

```text
JLCPCB_composants_BASIC_2026-09-11.xlsx
economic-parts.csv
source dataset : lrks/jlcpcb-economic-parts
snapshot : 351 composants library=base non supprimés
```

Références Basic utiles déjà identifiées :

```text
0 Ω                    C17477 / autres footprints Basic
4,7 Ω                  C17675
10 Ω                   C17415
22 Ω                   C17561
33 Ω                   C17634
47 Ω                   C17714
4,7 kΩ                 C25900
10 kΩ                  C25744
B5819W                  C8598
4,7 µF / 25 V          C1779
10 µF / 25 V           C15850
2,2 µF / 50 V X7R      C50254 (1206) ; C377773 existe en 0805 X5R
```

Les seules inductances `Basic` visibles dans le snapshot sont de petites inductances 10 µH à très faible courant et **ne conviennent pas** au TPS65132 ni au boost backlight.

Bias choisi :

```text
TPS65132WRVCT
LCSC/JLC : C1848345 lors de la dernière vérification
package : WQFN-20-EP, 3 × 4 mm
assemblage : compatible JLCPCB Economic + Standard ; Economic est imposé pour ce projet
```

Le `TPS65132A0YFFR` OEM Moto reste une référence électrique mais n’est plus le composant de production de l’interposer, car son DSBGA fin peut forcer le service Standard. Le `TPS65131RGER` n’est plus la cible non plus : sa BOM est nettement plus lourde.

Le nouveau design doit éviter de garder un composant complexe uniquement parce qu’il existait dans V1.

---

# 15. Modem LTE / téléphonie

[CIBLE RÉVISÉE — SOURCE DE VÉRITÉ EC25]

Le `SIMCom A7672E` reste abandonné. Le modem cible est un **Quectel EC25-EUX**, mais le matériel réellement acheté n'est pas un EC25 nu : il s'agit d'une **core board tierce `CC-MCore-EC25EUXGR` portant le EC25-EUX**.

```text
CC-MCore-EC25EUXGR
├── Quectel EC25-EUX
├── régulation/alimentation de la board
├── slot micro-SIM
├── connecteur USB
├── connecteurs antennes MAIN/AUX/GNSS
└── breakout de plusieurs signaux : UART, RI, DTR, PEN, RST, etc.
```

[CONFIRMÉ PAR OBSERVATION SUR L’EXEMPLAIRE REÇU — 2026-09-22]

Le marquage du module réellement reçu est :

```text
EC25-EUX
GA
EC25EUXGA-128-SGNS
```

Cette observation **supplante** le marquage `GR / EC25EUXGR-128-SGNS` vu précédemment sur la photo commerciale du vendeur.

Ne pas en déduire la version firmware : le marquage matériel ne donne pas `AT+QGMR`, l’état IMS ni la liste MBN. Ces informations restent à lire électriquement sur le module réel.

Rôle RF des trois connecteurs de la core board :

```text
MAIN  = antenne LTE principale, Tx/Rx
AUX   = antenne LTE secondaire/diversité ; pas une seconde connexion Internet
GNSS  = antenne satellite GNSS ; indépendante des antennes cellulaires
```

La localisation GNSS ne repose pas sur une triangulation des antennes cellulaires. Le récepteur calcule sa position à partir des signaux satellites ; les réseaux cellulaires/Wi-Fi peuvent seulement fournir une aide ou une localisation séparée.

Cette distinction est obligatoire :

```text
1. capacité du module EC25-EUX
2. capacité réellement routée/exposée par la core board
3. capacité réellement activée par le firmware installé
```

Ne jamais attribuer automatiquement à la core board toutes les interfaces du module brut.

## 15.0 Core board exacte et accès physique

[CONFIRMÉ PAR OBSERVATION]

- `RI` est accessible sur le header.
- `DTR` est accessible sur le header.
- la core board reçue expose visiblement un groupe sérigraphié `PCM` avec `IN`, `OUT`, `SYNC`, `CLK`, ainsi que `SDA`, `SCL`, `AD0`, `AD1`.
- l’autre rangée expose visiblement `VBUS`, `DN`, `DP`, `NET`, `GND`, `RXD`, `TXD`, `VIO`, `DTR`, `RI`, `BAT`, `GND`.
- la présence physique du breakout PCM/I²C réduit fortement l’intérêt d’un rework direct sur les pads LCC pour l’audio Minimal ; la continuité réelle vers les pins EC25 et les niveaux électriques doivent néanmoins être vérifiés avant connexion.
- les pads LCC situés sur le pourtour extérieur du EC25 restent physiquement accessibles pour soudure/rework ; seuls les pads cachés sous le module ne sont pas accessibles sans dessoudage.
- le slot SIM, le connecteur micro-USB et la partie alimentation apportés par la core board sont présents ; ils doivent être caractérisés plutôt que redessinés inutilement pour V1.
- trois connecteurs RF sont présents et sérigraphiés `DIV`, `GNS` et `MAIN` sur la board reçue ; `GNS` est traité comme le connecteur GNSS jusqu’à validation fonctionnelle.

[COREBOARD-VENDOR / À MESURER]

```text
VIN annoncé        5–40 V
VBAT annoncé       3,4–4,3 V
UART principal     3,3 V TTL annoncé
PEN                "Core board power enable"
RST                reset annoncé
Auto power-on      "Only support Auto power on"
```

Le schéma électrique exact de cette révision de core board n'est pas encore disponible. Les niveaux/polarités réels de `PEN`, `RST`, `RI`, `DTR` et leur éventuelle adaptation de niveau doivent être mesurés avant connexion directe à un GPIO MCU/Q6A.

## 15.1 Auto power-on, PEN, arrêt et recovery

L'indication commerciale **« Only support Auto power on »** signifie très probablement :

```text
alimentation appliquée à la core board
        ↓
EC25 démarre automatiquement
```

Le mécanisme exact reste [À VALIDER] : PWRKEY câblé, transistor/RC, relation avec `PEN`, ou autre logique de board.

Ne jamais conclure sans preuve :

```text
PEN = PWRKEY
```

`PEN` semble désigner **Power Enable** au niveau de la core board, pas une broche standard du EC25. Il faut déterminer si PEN coupe réellement le régulateur/rail du modem et quelle consommation subsiste lorsqu'il est désactivé.

Stratégie d'arrêt cible :

```text
ARRÊT NORMAL
MCU/Q6A -> AT+QPOWD ou séquence PWRKEY
       -> attendre arrêt propre du EC25
       -> couper PEN / rail si une coupure totale est souhaitée
```

```text
RECOVERY MODEM BLOQUÉ
coupure physique du rail / PEN si validé
       -> attente
       -> remise sous tension
       -> Auto power-on
```

Une coupure brute d'alimentation reste un mécanisme de récupération, pas le chemin normal d'arrêt. Quectel recommande un arrêt propre avant retrait du rail.

Le EC25 brut expose notamment :

```text
RESET_N  pin 20
PWRKEY   pin 21
```

`PWRKEY` direct n'est **pas un prérequis V1** si la core board auto-démarre et que PEN permet un vrai power-cycle. `RESET_N` est réservé au recovery, pas à l'arrêt normal.

## 15.2 Interfaces brutes utiles du EC25

Le pinout Quectel documente notamment :

```text
24 PCM_IN
25 PCM_OUT
26 PCM_SYNC
27 PCM_CLK
41 I2C_SCL
42 I2C_SDA
62 RI
66 DTR
67 TXD
68 RXD
69 USB_DP
70 USB_DM
71 USB_VBUS
```

Les signaux PCM/I²C et la logique du module nu sont dans le domaine **1,8 V**. Ne jamais appliquer directement du 3,3 V sur ces pads. Pour l'I²C codec, prévoir les pull-up adaptés vers 1,8 V.

Grâce à l'accès physique aux pads périphériques, il devient réaliste de reprendre directement uniquement les fonctions manquantes de la core board, en priorité :

```text
PCM + I²C pour codec audio externe
```

Priorité de reprise directe :

| Interface brute | Intérêt V1 | Décision |
|---|---:|---|
| PCM | élevé | récupérer si les pads 24–27 sont réellement accessibles |
| I²C codec | élevé | récupérer avec PCM si pads 41–42 accessibles |
| PWRKEY | moyen | seulement si PEN/auto-start ne suffisent pas |
| RESET_N | faible/moyen | recovery uniquement |
| RI | faible | déjà accessible sur la core board |
| DTR | faible | déjà accessible sur la core board |
| UART brut | faible | préférer le breakout 3,3 V si validé |
| USB D+/D− brut | à éviter | conserver le routage de la core board |
| SIM brut | inutile | slot déjà présent |
| RF brut | à éviter | utiliser les connecteurs antennes |

Ne pas dériver USB High-Speed sur des fils longs/stubs : D+/D− doivent rester dans un routage différentiel propre. Le connecteur USB de la core board reste le chemin préféré si son routage natif EC25 est confirmé.

## 15.3 Q6A / kernel — chemin USB cible

Q6A :

```text
modem Qualcomm interne désactivé au niveau MSS/GLINK
USB2 Host disponible
USB3 OTG/EDL conservé pour Android/debug
```

Le kernel Q6A exact contient déjà l'ID EC25 `2c7c:0125` dans `option.c` et `qmi_wwan.c`. Le défaut du build stock est surtout la configuration.

À intégrer built-in dans le kernel Maker Phone :

```text
CONFIG_USB_SERIAL_WWAN=y
CONFIG_USB_SERIAL_OPTION=y
CONFIG_USB_WDM=y
CONFIG_USB_NET_QMI_WWAN=y
```

[CONFIRMÉ PAR AUDIT SOURCE] **Aucun patch VID/PID ni driver Quectel externe n'est nécessaire pour reconnaître l'EC25.** Les archives Quectel `Option`/`QMI_WWAN` restent références comparatives, pas patches de production.

Data primaire :

```text
QMI/RmNet
```

Fallbacks seulement si nécessaire : ECM, éventuellement MBIM ; PPP dernier recours. `quectel-CM`, ModemManager et `libqmi` peuvent servir au diagnostic Linux mais ne doivent pas devenir une seconde pile de gestion QMI en production Android.

## 15.4 Android / Q6A ON — architecture Radio retenue

[CIBLE]

Connexion principale :

```text
Q6A USB2 Host -> core board -> EC25-EUX
```

Le lien USB est la voie normale pour :

```text
data LTE QMI/RmNet
ports série AT / URC / diagnostic
contrôle modem Android
audio UAC si le firmware l'expose
```

Le package Quectel retenu après audit des archives est :

```text
Quectel_Android_RIL_Driver_aidl3_V4.6
architecture ARM64
Radio AIDL V3
```

AOSP Android 15 `android-15.0.0_r36` possède les interfaces Radio AIDL V3 nécessaires.

Architecture :

```text
Android Telephony / Telecom / Connectivity
        ↓
rild AOSP dans vendor
        ↓
Quectel libril.so + libreference-ril.so (AIDL3 V4.6)
        ↓
AT/URC + QMI intégré au RIL
        ↓
EC25-EUX USB composite
```

Ne pas adapter QCRIL/QMI du modem Qualcomm interne au EC25.

### Fichiers vendor à intégrer

```text
/vendor/bin/hw/rild
/vendor/lib64/libril.so
/vendor/lib64/hw/libreference-ril.so
/vendor/etc/ql-ril.conf
Radio AIDL V3 NDK libraries nécessaires
```

Le package Quectel ne fournit pas le binaire `rild` ; la cible est le **`rild` AOSP Android 15 construit dans vendor**. `ql-ril.conf` est conservé sans options spéculatives tant que le hardware/firmware exact n'a pas été observé.

Instances VINTF cibles :

```text
IRadioConfig/default
IRadioData/slot1
IRadioMessaging/slot1
IRadioModem/slot1
IRadioNetwork/slot1
IRadioSim/slot1
IRadioVoice/slot1
```

Le produit EC25 ne doit pas exposer en parallèle les instances Radio HIDL Qualcomm concurrentes.

### Neutralisation de l'ancienne pile radio Qualcomm

À faire de façon ciblée :

```text
retirer ro.radio.noril=true
retirer ro.vendor.radio.noril=true
ne pas démarrer qcrilNrd / qcrild multi-SIM concurrents
retirer les déclarations Radio / RadioConfig HIDL concurrentes
ne pas utiliser qcrilhook / qtiradio / pile IMS/RTP Qualcomm pour l'EC25
conserver les composants Q6A non liés au modem externe (ex. Audio HAL native)
```

Retirer `noril` seul ne suffit pas : il faut aussi `rild`, blobs RIL, VINTF, dépendances, policy et accès devices.

### SELinux / ueventd

Réutiliser le domaine AOSP `rild` et appliquer le moindre privilège. Cible à valider au build/runtime :

```text
/vendor/bin/hw/rild -> rild_exec
/dev/ttyUSB*        -> radio/radio, accès ciblé rild
/dev/cdc-wdm*       -> accès QMI ciblé
wwan*               -> réseau / sysfs seulement si nécessaire
/dev/bus/usb/*      -> ne pas élargir globalement par défaut
```

Le `ueventd.rc` stock réserve au moins un `ttyUSB` à d'autres fonctions plateforme ; ne pas figer un numéro `ttyUSB2` ou autre avant capture réelle de l'arbre USB/sysfs.

Aucun `setenforce 0`, domaine permissif, `SYS_ADMIN` ou permission globale ne doit être utilisé comme solution finale.

### Data, SMS et appels

Le `libreference-ril.so` Quectel contient son propre chemin QMI :

```text
IRadioData/setupDataCall
  -> libril AIDL3
  -> libreference-ril
  -> /dev/cdc-wdmX + qmi_wwan
  -> wwanX
  -> DataCallResult
```

Les binaires RIL audités contiennent également les traitements SMS et voix :

```text
SMS  : CMGF, CMGL, CNMI, CNMA, QCMGS, IRadioMessaging, sendSms/sendImsSms
Voix : ATD, ATA, ATH, CLCC, CHLD, DTMF, IRadioVoice
```

Conclusion : **support logiciel présent**, mais compatibilité SIM/réseau/opérateur et audio doivent encore être prouvés sur le module réel.

### Telephony Q6A

Le vendor seul ne permet pas de certifier la présence complète de :

```text
com.android.phone
TelephonyProvider
CarrierConfig
Telecom
STK
features telephony
base APN
Dialer/Messaging
```

L'état reste **UNKNOWN** tant que `system`, `system_ext` et `product` ou un appareil complet ne sont pas inspectés. Ce n'est pas une incompatibilité démontrée.

## 15.5 IMS / VoLTE / MBN

[ÉTAT : SUPPORT MODULE OPTIONNEL + ARCHITECTURE PROBABLE, RÉSULTAT RÉEL À VALIDER]

Quectel annonce la VoLTE comme **optionnelle** pour la série/variante concernée. Ne jamais remplacer ce terme par « garanti ».

Le RIL Quectel contient notamment :

```text
AT+QCFG="ims"
AT+QIMSCFG
AT+QMBNCFG
getImsRegistrationState
imsNetworkStateChanged
sendImsSms
```

Aucun `ImsService` Android Quectel autonome ni dépendance à la pile IMS Qualcomm n'a été identifié dans le package audité.

Hypothèse la mieux supportée :

```text
EC25 firmware
  ├── IMS
  ├── MBN / profil opérateur
  └── signalisation VoLTE
        ↓
Radio HAL Quectel standard vers Android
```

Cette architecture est **probable**, pas encore prouvée sur notre firmware/SIM/opérateur.

Quatre conditions indépendantes :

```text
1. EC25-EUX capable
2. firmware installé capable/configuré
3. bon profil MBN présent et actif
4. SIM + opérateur acceptent l'enregistrement IMS
```

[CRITÈRE DE VALIDATION V1 / À PROUVER SUR MATÉRIEL]

Ne pas considérer le EC25 comme modem téléphonique final uniquement parce que la data LTE et les SMS fonctionnent. Le jalon téléphonie est un **vrai appel VoLTE sur SIM/opérateur cible**, avec :

```text
IMS enregistré
LTE maintenu pendant l’appel
audio duplex fonctionnel
appel entrant + sortant
DTMF / raccrochage corrects
```

Si ce jalon échoue durablement sur firmware/MBN/opérateur cible, réévaluer le modem final plutôt que dépendre d’un fallback 2G/3G.

Lecture sûre avant toute modification :

```text
ATI
AT+QGMR
AT+QCFG="ims"
AT+QMBNCFG="list"
```

Ne pas sélectionner/ajouter/supprimer de MBN avant sauvegarde complète des réponses et disponibilité d'une procédure de récupération.

La présence publique de profils VoLTE dans certains firmwares EC25-EUX ne prouve pas qu'ils existent dans **notre** firmware ni qu'un opérateur français donné acceptera le module.

## 15.6 Sleep réseau, DTR, RI et réveil Q6A

[CONFIRMÉ AU NIVEAU MODULE / CIBLE POUR LE TÉLÉPHONE]

Le EC25 possède un **Sleep Mode** qui réduit la consommation tout en conservant la capacité à recevoir les événements réseau nécessaires à un téléphone (paging, SMS, appel voix et données selon configuration).

Ne pas confondre ce mode avec :

```text
AT+CFUN=0
```

`CFUN=0` désactive des fonctions radio/SIM et ne convient pas au besoin « téléphone joignable en veille ».

Cible via UART :

```text
AT+QSCLK=1
DTR = HIGH / état autorisant le Sleep
        ↓
EC25 peut dormir lorsqu'aucune activité ne le retient éveillé
```

Réveil explicite par le host :

```text
DTR = LOW
```

`RI` peut signaler une URC/événement entrant et servir d'interruption de réveil pour le MCU. Quectel documente un comportement configurable de RI ; la forme exacte des impulsions pour notre firmware doit être mesurée.

Architecture cible :

```text
EC25 alimenté + enregistré + Sleep
        │
        ├── DTR piloté par MCU
        └── RI surveillé par MCU
                 ↓
          appel / SMS / URC
                 ↓
                RI
                 ↓
               MCU
          ┌──────┴──────┐
          │             │
 traitement Minimal   réveil/allumage Q6A
```

Cela devient un axe prioritaire pour l'autonomie.

### Q6A suspendu

C'est le cas le plus simple : USB doit entrer proprement en suspend. Le remote wake USB peut éventuellement être utilisé si le chemin Q6A le supporte ; sinon `RI -> MCU_IRQ/wake` reste la voie robuste.

### Q6A totalement OFF

C'est compatible avec le concept Minimal Phone, mais [À VALIDER] électriquement :

```text
USB_VBUS réellement coupé ?
back-power via USB/ESD ?
D+/D− laissent-ils EC25 dormir ?
RI reste-t-il actif ?
UART MCU reste-t-il opérationnel ?
```

Le schéma final peut nécessiter un contrôle explicite de `USB_VBUS` / switch de connexion USB entre Q6A et core board.

## 15.7 Mode Minimal Phone / Q6A OFF

[CIBLE RÉVISÉE]

Lorsque le Q6A est totalement hors tension, le MCU prend la maîtrise fonctionnelle du modem par UART :

```text
MCU <-> UART <-> EC25
MCU -> DTR / commandes AT de veille
EC25 RI -> MCU
MCU -> PEN/PWRKEY uniquement après validation électrique
```

Fonctions visées sans Q6A :

```text
réception d'appels
émission d'appels
SMS
état réseau
mini-display
réveil du Q6A à la demande
```

Politique de propriété :

```text
Q6A ON  : Android/Q6A est maître fonctionnel du modem via USB
Q6A OFF : MCU est maître fonctionnel du modem via UART
```

Éviter toute émission AT concurrente contradictoire entre Android et MCU lorsque les deux sont actifs.

Le mode Minimal doit idéalement garder l'EC25 en Sleep entre événements plutôt que pleinement actif en permanence.

## 15.8 Protocole de validation EC25 avant intégration Android

Avant de flasher une image Android modifiée, brancher la core board sur un PC Linux et établir le comportement réel du matériel.

Ordre sûr :

```text
1. photo HD / révision PCB / mapping pads
2. mesure niveaux RI/DTR/PEN/RST/UART sans connexion GPIO externe
3. lsusb / lsusb -v / usb-devices / dmesg
4. mapper ttyUSB*, cdc-wdm*, wwan* via sysfs
5. ATI / AT+CGMM / AT+CGMR / AT+QGMR
6. AT+CPIN? / AT+CSQ / AT+CEREG? / AT+COPS?
7. AT+QCFG="usbnet"
8. AT+QCFG="usbcfg"
9. AT+QCFG="ims"
10. AT+QMBNCFG="list"
11. test QSCLK + DTR + consommation + RI sur SMS/appel
12. test QMI data
13. test SMS MO/MT
14. test appels MO/MT + DTMF
15. test VoLTE : IMS + maintien LTE pendant appel
16. test UAC si support firmware
17. PEN/RST seulement en dernier
```

Masquer IMEI/IMSI/ICCID dans les rapports publiés.

---

# 16. Audio

[CIBLE RÉVISÉE]

## 16.1 Audio Q6A / Android

Le Q6A conserve son codec audio natif pour tout le fonctionnement Android.

```text
J17 = jack TRRS utilisateur
J18 = audio interne Android : EAR / AUX / AMIC / MIC_BIAS
```

Décision :

```text
jack casque -> chaîne native Q6A / J17
écouteur interne -> J18 EAR
haut-parleur -> J18 AUX -> ampli Class-D
micro Android -> J18 AMIC
micro casque -> J17
```

**Ne pas ajouter de DAC externe au Q6A uniquement pour alimenter le haut-parleur.** Le codec Q6A fournit déjà les sorties analogiques nécessaires ; ajouter un nouveau codec numérique augmenterait inutilement l'intégration kernel/audio.

Le jack J17 est prévu comme jack Android natif. S'il doit être mécaniquement déporté, conserver proprement les signaux casque/micro/détection ; ne pas créer une rallonge qui force artificiellement la détection de casque en permanence.

## 16.2 Audio EC25 -> Q6A en mode Android

[CIBLE / À VALIDER SUR FIRMWARE]

Premier choix :

```text
EC25 -> USB Audio/UAC -> Q6A -> codec Q6A -> jack / écouteur / HP
```

Cette voie permet de conserver une seule chaîne audio physique côté Android et évite un chemin analogique modem->Q6A.

Le Q6A possède déjà le support USB Audio côté kernel et un HAL audio USB dans le vendor audité. Le verrou est donc surtout le **firmware EC25 exact** et sa composition USB.

Procédure à valider sur le modem réel :

```text
AT+QCFG="USBCFG"   -> lire et sauvegarder la composition actuelle
```

Si le champ UAC est réellement disponible, suivre uniquement la procédure Quectel correspondant au firmware, avec sauvegarde/recovery prête, puis reboot/power-cycle et configuration PCM-over-USB (`AT+QPCMV=1,2` dans la note concernée).

Si le champ UAC n'existe pas sur notre firmware :

```text
UAC_NOT_AVAILABLE_WITH_CURRENT_FIRMWARE
```

Ne pas modifier arbitrairement VID/PID ou supprimer les fonctions AT/QMI nécessaires au recovery.

Un périphérique UAC qui s'énumère ne prouve pas encore l'audio d'appel : tester séparément capture, playback, appel duplex et VoLTE.

## 16.3 Audio Minimal Phone / fallback indépendant de UAC

[CIBLE RÉVISÉE]

Le mode Minimal Phone doit continuer à permettre un appel vocal alors que le Q6A est **physiquement hors tension**. L'audio ne peut donc pas dépendre du codec Q6A dans ce mode.

Voie cible :

```text
EC25 PCM 1,8 V -> codec audio Minimal dédié -> écouteur / ampli HP / micro LTE
```

Le EC25 expose :

```text
24 PCM_IN
25 PCM_OUT
26 PCM_SYNC
27 PCM_CLK
41 I2C_SCL
42 I2C_SDA
```

Nouvelle information matérielle : les pads périphériques du EC25 restent accessibles sur la core board. Il faut vérifier la position physique exacte de 24–27 et 41–42 ; s'ils sont sur le pourtour accessible, un petit rework/flex peut récupérer PCM + I²C même si le header de la core board ne les expose pas.

Tous ces signaux doivent être traités dans leur domaine **1,8 V** ; l'I²C nécessite des pull-up adaptés vers 1,8 V. Ne pas brancher directement des GPIO 3,3 V.

`ALC5616` reste le **premier candidat** parce que Quectel documente explicitement cette association et `AT+QDAI=3`. Il n'est pas encore figé pour la BOM finale : stock, package, consommation, niveaux et intégration JLC doivent être vérifiés.

Autres codecs explicitement supportés par des commandes EC25 historiques : `NAU8814` et `TLV320AIC3104`. Ne pas en choisir un sans vérifier la documentation/firmware de notre EC25-EUX exact.

Le MCU ne transporte **jamais** les échantillons audio. Il contrôle l'état du téléphone et le modem ; le flux vocal reste entre EC25, codec et transducteurs.

Stratégie générale :

```text
Android/Q6A ON : UAC en priorité si validé
Minimal/Q6A OFF: PCM + codec dédié
Fallback Android si UAC impossible : PCM/codec peut aussi rester disponible selon le routage final
```

## 16.4 Partage des transducteurs

Pour partager le même écouteur et le même haut-parleur entre Android et Minimal Phone, utiliser des switchs/mux analogiques appropriés avant les étages concernés.

```text
Q6A J18 EAR  ou codec Minimal -> switch -> écouteur
Q6A J18 AUX  ou codec Minimal -> mux -> ampli Class-D -> haut-parleur
```

Ne jamais sommer directement deux sorties audio actives ou deux sorties différentielles/BTL.

Pour V1, conserver deux micros séparés est préféré :

```text
micro Android -> Q6A J18 AMIC
micro Minimal/LTE -> codec EC25
micro casque -> J17
```

Cela évite les problèmes de MICBIAS, d'impédance et de commutation entre deux codecs.

Le jack J17 n'est **pas requis en Minimal Phone V1**.

Minimal Phone avec Q6A OFF :

```text
écouteur interne : oui
haut-parleur : oui
micro LTE dédié : oui
jack J17 : non
```

Composants exacts des mux/switchs/ampli Class-D et du codec Minimal : [À VALIDER].

---

# 17. Caméras / autres modules V1

```text
Caméra avant :
  module USB UVC OV5693 compact
  USB2 Host #2
  dimensions réelles à valider

Caméra arrière :
  Radxa Camera13M 214 / Sony IMX214
  CSI 4 lanes officiel
  IMX577 = alternative non retenue actuellement

Haptique :
  moteur 8×2 mm
  driver/MOSFET à prévoir

ALS/proximité :
  nécessaires, références non choisies

IMU :
  à vérifier selon ce qui est déjà exploitable sur Q6A

NFC/fingerprint :
  V2 / non prioritaires
```

USB internes V1 :

```text
Host #1 EC25-EUX carrier
Host #2 caméra avant OV5693
Host #3 réserve
```

---

# 18. Maker Port

[CIBLE]

Ne jamais exposer directement les GPIO Q6A au monde extérieur.

Prévoir :

```text
level shifting 1,8 ↔ 3,3/5 V
buffers
résistances série / limitation courant
TVS / clamps
protection surtension
ADC externe si utile
mux analogique éventuel
connecteur + breakout
```

Fonctions possibles :

```text
GPIO
SPI
I²C
UART
PWM
ADC/voltmètre
logic probe
data logger
LoRa
CAN/RS485
SDR
JTAG/SWD
```

Références non gelées.

---

# 19. Thermique / mécanique

[CIBLE V1]

Puissance visée :

```text
~3–4 W soutenus
6–8 W boost court
throttling au-delà
```

Architecture :

```text
Q6A hotspot -> TIM -> cuivre 0,3–1 mm -> grande plaque Al 1060 extérieure

rupture thermique

batterie 2S -> interface douce -> petite plaque Al 1060 extérieure
```

Coque principale :

```text
bois
deux inserts aluminium 1060 ~1 mm affleurants
joint périphérique
pas de conduit d'air traversant
```

Retirés V1 :

```text
ventilateur interne
vapor chamber
graphite obligatoire
```

Cooler externe possible V2/test.

Ne pas thermiquement encapsuler totalement la batterie.

## 19.1 Proposition — supervision thermique de coque par le MCU

[PROPOSITION / À VALIDER — **NON FIGÉE**]

Ajouter une sonde de température dédiée à la coque / à l’insert aluminium, lue par le MCU always-on. L’objectif n’est pas de remplacer les protections thermiques internes du Q6A ni celles du pack batterie, mais d’ajouter une mesure représentative de la température réellement ressentie à l’extérieur du téléphone.

Principe proposé :

```text
sonde coque -> MCU
MCU -> état thermique / événement -> Q6A via liaison MCU<->Q6A + IRQ
Q6A/Linux -> politique de limitation CPU/GPU/système
```

Le MCU ne devrait pas commander directement une fréquence CPU précise. Il remonterait plutôt un niveau abstrait (ex. NORMAL / WARM / HOT / CRITICAL) ou une température mesurée. La politique exacte de performance resterait gérée côté Linux/Android, afin de pouvoir faire évoluer les limites CPU/GPU sans modifier le firmware MCU.

Intégration logicielle préférée : exposer la mesure ou l’état MCU comme une vraie zone thermique Linux de type `skin` / `case`, puis laisser le framework thermal Linux et la couche thermique Android appliquer les mécanismes de mitigation. Éviter un polling permanent dans une application Android utilisateur.

Comportements possibles, à valider expérimentalement :

```text
WARM      -> suppression/réduction des boosts
HOT       -> mode éco marqué, plafonds CPU/GPU
CRITICAL  -> limitation sévère + demande d’arrêt propre
EMERGENCY -> si Q6A bloqué et température continue de monter : coupure matérielle après timeout de sécurité
```

Le MCU pourrait aussi surveiller ultérieurement d’autres températures externes (batterie, zone USB-C/chargeur, modem), mais aucune de ces extensions n’est figée.

Points à valider avant gel :

```text
type exact de sonde (NTC ou capteur numérique)
emplacement mécanique représentatif de la température de coque
seuils réels après mesures sur prototype
protocole MCU -> Q6A et niveaux d’état
intégration kernel / thermal zone / HAL Android exacte
interaction avec les protections thermiques internes Qualcomm
politique d’arrêt coopératif puis hard-cut de secours
```

Cette proposition est **complémentaire** au contrôle de puissance/rush-to-idle envisagé côté Q6A ; elle n’est pas une décision figée à ce stade.

---

# 20. Bring-up global recommandé

Ordre :

```text
1. sauvegarde/rollback Android + partitions
2. valider alimentation 2S / rails / protections sans écran
3. valider Q6A stock
4. valider kernel reconstruit IDENTIQUE fonctionnellement avant ajout touch
5. interposer sans dalle : courts-circuits, 1,8 V et 5 V
6. programmer TPS65132WRVCT via les 4 pads avec Pico : +5,5/-5,7 V -> NVM -> power-cycle -> verify PASS
7. valider ENP/ENN et mesurer +5,5/-5,7 V sans dalle
8. valider les 74LVC2T45 SPI/RESET/IRQ
9. sur Q6A : retirer R315 1,1 Ω avant tout essai Moto ; contrôler C644 physiquement
10. poser RSET initial ~8,2 Ω et vérifier le réseau de straps : une seule sélection active
11. brancher dalle avec backlight OFF
12. DSI 60 Hz + DSC
13. reset + DCS complet
14. vérifier 0x0A == 0x9C
15. backlight très faible : mesurer ΔV sur chacun des 3 ballast 47 Ω -> I=ΔV/47
16. vérifier équilibre K1/K2/K3 et VCC_LEDA ; arrêter si déséquilibre/OVP
17. seulement ensuite réduire RSET progressivement si nécessaire
18. tactile SPI 4,8 MHz : lire trim ID AVANT firmware
19. firmware xpeng seulement si ID compatible
20. vérifier input
21. tactile 9,6 MHz
22. display 90 -> 120 -> 144 -> DFPS

23. AVANT Android modifié : inspecter la core board EC25, relever révision et pads périphériques accessibles
24. mesurer RI/DTR/PEN/RST/UART sur la core board ; aucun GPIO MCU/Q6A direct avant validation des niveaux
25. PC Linux : lsusb / lsusb -v / usb-devices / dmesg ; confirmer que le connecteur expose l'USB composite natif EC25
26. mapper réellement ttyUSB*, cdc-wdm*, wwan* ; ne pas figer ttyUSB2 par hypothèse
27. lire firmware/config sans modification : ATI, CGMM/CGMR/QGMR, usbnet, usbcfg, ims, QMBNCFG=list
28. tester EC25 Sleep : AT+QSCLK=1 + DTR ; mesurer consommation
29. générer SMS/appel entrant et mesurer RI ; valider RI -> MCU wake
30. valider DTR bas -> réveil explicite EC25
31. valider QMI/RmNet sur PC, puis SMS MO/MT et signalisation appels MO/MT
32. valider VoLTE seulement si IMS/MBN le permettent : état IMS + LTE maintenu pendant appel + audio duplex
33. valider UAC sur PC uniquement si le firmware expose l'option ; sauvegarder USBCFG avant toute écriture
34. si UAC insuffisant : vérifier pads PCM 24–27 + I²C 41–42 accessibles et préparer fallback codec
35. tester PEN/Auto power-on/RST en dernier ; distinguer shutdown propre, reset et hard power-cycle
36. rebuild kernel Maker Phone : touch + USB_SERIAL_WWAN/OPTION/WDM/QMI_WWAN built-in
37. intégrer vendor Radio : rild AOSP + Quectel AIDL3 V4.6 + dépendances Radio AIDL V3 + VINTF + SELinux/ueventd
38. retirer noril et neutraliser uniquement les services/manifests Radio/IMS Qualcomm concurrents
39. checkvintf + build sepolicy enforcing + vérification deps ELF avant flash
40. sur Android : enumeration EC25, Radio AIDL, SIM, data DataCallResult, SMS, appels
41. valider UAC EC25 -> Q6A si firmware exact compatible
42. valider MCU <-> EC25 par UART + RI + DTR ; PEN/PWRKEY seulement selon résultat électrique
43. valider Q6A suspendu : EC25 Sleep + RI -> réveil Q6A
44. valider Q6A totalement OFF : EC25 Sleep + MCU actif ; vérifier USB_VBUS/back-power avant de figer le schéma
45. valider MCU <-> Q6A : I²C + IRQ + événements input
46. valider démarrage Q6A par MCU puis shutdown coopératif + SAFE_TO_CUT
47. valider coupure physique Q6A sans corruption après arrêt propre
48. valider audio Android : J17 + J18 EAR/AUX + ampli HP
49. valider audio Minimal : EC25 PCM -> codec dédié -> écouteur/HP/micro
50. valider bascule Android <-> Minimal
51. valider suspend/resume/power states complets
```

---

# 21. Points encore ouverts avant PCB final

**Règle de périmètre :** la PCB actuellement à finaliser est uniquement l’interposer écran minimal de la section 12. Les rubriques Power board, MCU, modem, audio et mécanique ci-dessous restent des chantiers système séparés et **ne doivent pas être absorbées dans cette PCB** sans nouvelle décision explicite.

Priorité élevée :

```text
mating/orientation exact connecteur Q6A J10
identité physique de l'écran aftermarket
GPIO/pinctrl final RESET et TE display
implémentation normale ENP/ENN du TPS65132WRVCT et séquence retenue
MPN/footprint exact inductance 2,2 µH du TPS65132WRVCT
réalisation finale du petit jig Pico / adaptation de niveau I²C
nouveau schéma interposer simplifié + ERC/layout
modification Q6A SY7203 : accessibilité physique R315/C644
rating réel de C644 monté [À VALIDER SUR MATÉRIEL]
valeurs RSET supplémentaires après le premier essai 8,2 Ω
mesures partage courant K1/K2/K3 sur les 6 TP ballast
AVB/repack final avec procédure rollback
```

Power board :

```text
choix définitif PD/charger/gauge/eFuse
rail EC25 / alimentation carrier et budget courant
rail 5 V système / budget courant
power-path USB-C / batterie
```

MCU / power supervision :

```text
choix du bus I²C Q6A libre pour le MCU
GPIO MCU_IRQ et éventuel SAFE_TO_CUT
commande PWR Q6A par open-drain
switch/load-switch principal Q6A et logique de coupure
driver kernel input MCU + protocole de registres/FIFO
comportement bootloader/recovery ; garder 2 GPIO de secours si possible
validation consommation Pico puis choix éventuel MCU final low-power
proposition thermique MCU : sonde coque, protocole d’état, seuils et intégration thermal Linux/Android [À VALIDER]
```

Modem / EC25 :

```text
schéma/révision exacte CC-MCore-EC25EUXGR si disponible
firmware exact EC25-EUX
VID/PID + descripteur USB + mapping ttyUSB/cdc-wdm/wwan
mode data QMI/RmNet réel
AIDL3 V4.6 : build rild/VINTF/SELinux/ueventd et dépendances Radio V3
Telephony complet dans system/product/system_ext
IMS réel + liste/activation MBN + compatibilité opérateur/SIM VoLTE
validation UAC sur firmware exact
validation QSCLK Sleep + DTR + RI sur SMS/appel
UART MCU <-> carrier
niveaux électriques RI/DTR/PEN/RST/UART
fonction exacte de PEN et consommation PEN OFF
Auto power-on : mécanisme réel
contrôle USB_VBUS / risque back-power quand Q6A OFF
accessibilité réelle des pads PCM 24–27 / I²C 41–42 si reprise directe nécessaire
comportement arrêt propre AT+QPOWD puis coupure rail
```

Audio :

```text
codec Minimal exact ; ALC5616 candidat
switchs/mux écouteur
ampli Class-D
niveaux EAR/AUX Q6A
niveaux codec Minimal
micros séparés Android / Minimal
```

Mécanique/RF :

```text
antennes
fenêtres RF
coque
acoustique
batteries réelles
thermique réel
```

---

# 22. Documents disponibles — à demander au propriétaire si non présents

## 22.1 Sources principales

```text
Q6A_Android15_20260630_inspection_handoff_v10(4).md
```

Handoff historique massif. Source de contexte global, firmware, power, modem, audio, thermal, BOM.  
**Sections 100/101/102 = ancienne source de vérité globale ; ce master la supersède sur display/touch/interposer.**

```text
SCHEMATIC_FROZEN_V1(5).md
```

Freeze de l'ancien interposer V1 : TPS61194 + TPS65131 + 4×74LVC2T45 + 3×TPD4E05U06. **Historique seulement : TPS61194/TPS65131/TPD4E05U06 ne font plus partie de la cible.**

```text
V1 (2)(1).zip
```

Projet KiCad historique : `V1.kicad_sch`, `V1.kicad_pcb`, projet/feuilles hiérarchiques/libs. À utiliser pour récupérer structure/symboles, pas comme design cible final.

## 22.2 Schémas constructeur

```text
radxa_dragon_q6a_schematic_v1.21.pdf
```

Source primaire Q6A : J10, GPIO/QUP, translators, power, native backlight, audio, USB, etc.

```text
radxa_dragon_q6a_components_placement_map_v1.21.pdf
```

Placement composants Q6A ; utile pour modifications physiques R315/C644/U43/backlight.

```text
772288625-XT2175-x-Yukon-Moto-g200-5G-MB-Schematics-L3-Repair(2).pdf
```

Schéma Moto G200 : connecteur display, DSI, bias, AW99703 backlight, tactile, protections.

## 22.3 Rapports finaux récents

```text
Q6A_NOVATEK_FINAL_INTEGRATION_REPORT.md
```

**Source de vérité touch logiciel.** Conclusion D1 : kernel Q6A reconstruit + driver Novatek SPI minimal intégré.

```text
Q6A_NT36672E_FINAL_DISPLAY_REPORT.md
```

**Source de vérité display logiciel.** Conclusion A : DTBO-only, aucun patch kernel display.

## 22.4 Missions intermédiaires — historique de méthode

Ne pas traiter comme résultats finaux :

```text
MISSION_Q6A_NOVATEK_TOUCH_PORTING.md
MISSION_Q6A_NOVATEK_PHASE2_VALIDATION.md
MISSION_Q6A_NOVATEK_PHASE3_MODULE.md
MISSION_Q6A_NOVATEK_FINAL.md
MISSION_Q6A_DISPLAY_FINAL.md
```

Elles sont utiles seulement pour reproduire/auditer les recherches.

## 22.5 Fichiers existant dans le workspace utilisateur mais pas forcément joints au chat

Demander à l'utilisateur si nécessaires :

```text
Q6A_NOVATEK_PHASE3_MODULE_REPORT.md
evidence/
display_evidence/
```

`display_evidence/` contient notamment selon le rapport :

```text
q6a-entry40-nt36672e.dts
q6a-entry40-nt36672e.dtbo
analyses DSC/DFPS
round-trip DTBO
matrice propriétés Moto -> Q6A
```

## 22.6 JLCPCB

```text
JLCPCB_composants_BASIC_2026-09-11.xlsx
economic-parts.csv
jlcpcb-economic-parts-pages.zip
```

Utiliser pour privilégier JLC Basic/Preferred.

## 22.7 EC25 / Quectel — rapports et paquets audités

Rapports consolidés disponibles :

```text
EC25_Q6A_MASTER_INTEGRATION_REPORT(4).md
EC25_Q6A_MASTER_INTEGRATION_REPORT(5)_UPDATED.md
```

Le présent PROJECT_STATE intègre leurs conclusions et prime désormais sur eux en cas de divergence.

Archives Quectel acquises/auditées :

```text
Quectel_Android_RIL_Driver_aidl1_V4.6
Quectel_Android_RIL_Driver_aidl2_V4.7
Quectel_Android_RIL_Driver_aidl3_V4.6
Quectel_Linux_Android_QMI_WWAN_Driver_V1.3
Quectel_Linux_USB_Serial_Option_Driver_V1.1
```

Cible retenue : **AIDL3 V4.6 ARM64**. Les drivers Linux Quectel ne sont pas injectés dans le kernel Q6A : le kernel exact possède déjà les IDs EC25.

Documents Quectel à conserver/rechercher avec priorité : Hardware Design récent, Reference Design, AT Manual, QCFG, USB Descriptor, IMS Application Note, Voice Over USB/UAC, UAC Application Note, WCDMA&LTE Audio Design Note, Low Power Mode, Data Call, PCB Design Guide et release notes du firmware EC25-EUX exact.

---

# 23. Sources publiques importantes

Radxa :

```text
https://docs.radxa.com/en/dragon/q6a/
https://docs.radxa.com/en/dragon/q6a/low-level-dev/build-system/kernel
https://docs.radxa.com/en/dragon/q6a/hardware-use/mipi-dsi
https://docs.radxa.com/en/accessories/display/lcd-10-fhd/lcd-10-fhd-product
https://dl.radxa.com/accessories/10-fhd/radxa_display_10_fhd_product_brief.pdf
https://dl.radxa.com/dragon/q6a/
https://dl.radxa.com/q6a/hw/radxa_dragon_q6a_schematic_v1.21.pdf
```

Bias LCD :

```text
https://www.ti.com/product/TPS65132
https://www.ti.com/lit/ds/symlink/tps65132.pdf
```

Backlight Q6A :

```text
https://www.silergy.com/productsview/SY22104DBC
https://datasheet.lcsc.com/lcsc/1804162155_Silergy-Corp-SY7203DBC_C125894.pdf
```

Motorola/Lineage :

```text
https://github.com/LineageOS/android_kernel_motorola_sm7325
https://github.com/LineageOS/android_device_motorola_xpeng
```

Firmware propriétaire xpeng :

```text
https://github.com/TheMuppets/proprietary_vendor_motorola_xpeng
```

Firmware tactile :

```text
proprietary/vendor/firmware/novatek_ts-NT36675-21101302-6044-xpeng.bin
```

Quectel EC25 :

```text
https://www.quectel.com/product/lte-ec25-series/
https://quectel.com/content/uploads/2024/02/Quectel_EC25_Series_Hardware_Design_V2.4-4.pdf
```

Documents Quectel prioritaires :

```text
Quectel_EC25_Series_Hardware_Design (version récente si disponible, ex. V2.8)
Quectel_EC25_Series_Reference_Design
Quectel_EC25_Series_LTE_Standard_Module_Specification
Quectel_EC2x&EG2x&EG9x&EM05_Series_AT_Commands_Manual
Quectel_EC2x&EG2x&EG9x&EM05_Series_QCFG_AT_Commands_Manual
Quectel_EC2x&EG2x&EG9x&EM05_Series_USB_Descriptor_Introduction
Quectel_EC2x&EG2x&EG9x&EM05_Series_IMS_Application_Note
Quectel_LTE_Standard_UAC_Application_Note
Quectel_EC2x_EG9x_Voice_Over_USB_and_UAC_Application_Note
Quectel_WCDMA&LTE_Audio_Design_Note
Quectel_EC2x&EG2x&EG9x&EM05_Series_Low_Power_Mode_Application_Note
Quectel_EC2x&EG2x&EG9x&EM05_Series_Data_Call_Application_Note
Quectel_EC2x&EG2x_Series_PCB_Design_Guide
firmware release notes EC25-EUX exact
```

Points EC25 à conserver comme références :

```text
Android RIL cible      : Quectel AIDL3 V4.6 ARM64 / Radio AIDL V3
Data primaire          : QMI/RmNet
Kernel Q6A             : Option + WDM + QMI_WWAN built-in, sans driver externe Quectel
AT+QDAI=1              : PCM configurable
AT+QDAI=3              : profil ALC5616 documenté
AT+QSCLK=1             : autorise Sleep selon conditions DTR/USB
DTR                     : contrôle sommeil/réveil selon configuration
RI                      : signal de réveil/URC exploitable par MCU
AT+QCFG="ims"           : état/config IMS à inspecter
AT+QMBNCFG="list"       : profils MBN à lire avant toute modification
AT+QCFG="USBCFG"        : composition USB/UAC à sauvegarder avant modification
UAC                     : à valider sur le firmware EC25-EUX exact
VoLTE                   : optionnelle, dépend firmware + MBN + SIM + opérateur
PCM/I²C bruts           : domaine 1,8 V
```

---

# 24. Anciennes décisions explicitement SUPPLANTÉES

Ne pas restaurer ces choix sans nouvelle justification :

| Ancienne décision | État courant |
|---|---|
| TPS61194 sur interposer | remplacer par SY7203 natif Q6A + 3×47 Ω + mesure individuelle |
| TPS65131RGER pour bias | **remplacé par TPS65132WRVCT WQFN-20** |
| TPS65132A0YFFR DSBGA sur notre interposer | référence OEM seulement ; rejeté si cela force Standard PCBA |
| Standard PCBA comme choix par défaut | rejeté : **la carte doit rester en Economic PCBA** |
| bias symétrique TPS65132B5 ±5,5 V | rejeté : conserver +5,5/-5,7 V Moto via TPS65132W programmable |
| connecteur de programmation bias | aucun connecteur : 4 pads SMD 2,54 mm GND/SDA/SCL/+5V |
| programmation bias à chaque boot | inutile : programmer une fois et stocker en EEPROM |
| 3× TPD4E05U06 sur DSI | supprimer sur interposer interne |
| TXS0108E/TXB0108 comme solution finale SPI | non retenu : conserver 74LVC2T45 directionnels |
| RSET ~4,1 Ω / ~49 mA total au premier test | trop agressif : premier essai ~8,2 Ω / ~24,4 mA total |
| C644 10 V forcément à remplacer | d’abord vérifier le composant réel ; remplacer seulement si rating insuffisant |
| GPIO tactile non affectés | SPI6 QUP0_SE6 mappé au header |
| tactile via driver NT36xxx stock Q6A | impossible : stock = I²C |
| tactile via `.ko` externe comme voie principale | rejeté : MODVERSIONS/ABI non reproductible |
| full Android rebuild | non nécessaire |
| patch kernel display | non nécessaire |
| driver AW99703 Moto | non nécessaire pour le premier prototype |
| 2N7002 simple pour SPI tactile | ne pas utiliser comme solution SPI générique |
| boost 2S -> 12 V Q6A | supprimé |
| SIMCom A7672E comme modem V1 | **remplacé par Quectel EC25-EUX sur carrier prototype** |
| modem QCRIL/QMI Qualcomm adapté au SIMCom | rejeté ; modem externe EC25 piloté par couche Radio HAL/service dédiée |
| audio analogique direct du SIMCom vers mux téléphone | remplacé : Android via USB/UAC si validé ; Minimal via PCM + codec dédié |
| boutons câblés en parallèle MCU + Q6A | rejeté : boutons physiques uniquement sur MCU ; événements Q6A via I²C/IRQ, PWR matériel séparé |
| MCU comme simple auxiliaire boutons/watchdog | remplacé : **MCU = superviseur matériel always-on du téléphone** |
| DAC externe dédié au haut-parleur Q6A | rejeté : utiliser J18 EAR/AUX + ampli Class-D ; codec supplémentaire réservé au Minimal Phone |
| ventilateur interne V1 | supprimé |

---

# 25. État synthétique par sous-système

| Sous-système | État |
|---|---|
| Q6A / compute | **VALIDÉ SUR MATÉRIEL : Q6A V1.21, EDL Qualcomm opérationnel, flash QSPI/eMMC réussi, Android stock booté** |
| Android 15 image | **AUDITÉE + FLASHÉE + BOOT RÉUSSI ; archive stock SHA-256 `1978e216...ebb16` ; baseline matérielle établie le 2026-09-22 ; ADB/logs et inventaire téléphonie encore à capturer avant mods** |
| Kernel exact | IDENTIFIÉ |
| Display NT36672E | **DTBO-only CONFIRMÉ** |
| Touch Novatek | **kernel rebuild + driver SPI minimal = stratégie courante ; alternative RP2040 -> USB HID à prototyper, non figée** |
| DSI matériel | direct, 4 lanes, redesign sans TVS discrètes |
| Touch SPI matériel | SPI6 3,3 V -> **4×74LVC2T45** -> 1,8 V |
| Bias LCD | **TPS65132WRVCT WQFN-20 ; NVM +5,5/-5,7 V ; Economic PCBA** |
| Programmation bias | **4 pads SMD 2,54 mm + Pi Pico USB/PC** |
| Backlight | **SY7203 natif Q6A + 3×47 Ω + 6 TP + RSET selectable ; 8,2 Ω premier essai** |
| Modem | **core board tierce + EC25-EUX réel marqué `GA / EC25EUXGA-128-SGNS` ; breakout RI/DTR + PCM/I²C visible ; QMI/RmNet + Quectel RIL AIDL3 V4.6 ; niveaux électriques, Sleep, VoLTE/UAC/firmware à valider** |
| Audio | **Q6A natif pour Android ; UAC premier choix si firmware compatible ; PCM/I²C 1,8 V récupérables sur pads périphériques comme fallback/Minimal ; codec/mux/ampli à choisir** |
| Batterie | 2S validée |
| Charge / PD | architecture validée ; ICs encore candidats |
| MCU always-on | **superviseur matériel ; Pico/RP2040 validé pour prototype ; MCU final low-power à choisir** |
| Caméras | choix V1 défini |
| Thermique | architecture V1 définie ; **supervision température de coque par MCU proposée, non figée** |
| Maker Port | concept validé ; électronique à concevoir |
| PCB interposer écran | **PÉRIMÈTRE FIGÉ : PCB minimale écran/adaptation uniquement ; SCHÉMA/LAYOUT À REDESSINER** |
| Matériel Q6A réel | **REÇU ET INSPECTÉ : Dragon Q6A V1.21 + eMMC YMTC 64 GB ; accès EDL/Firehose validé ; Android stock booté avec succès le 2026-09-22** |

---

# 26. Directive à tout futur agent

1. Lire ce document en premier.
2. **Gel de périmètre PCB : la carte à concevoir maintenant est une PCB minimale dédiée uniquement à l’écran/tactile et à leur adaptation vers le Q6A. Ne pas y intégrer MCU, USB-C/charge, Maker Port, modem/audio, haptique, capteurs ou autres fonctions téléphone sans nouvelle décision explicite.**
3. Les réflexions de regroupement en « Main Board » sont **en suspens** et ne constituent pas une architecture de fabrication actuelle.
4. Ne pas repartir du schéma V1 comme si ses ICs étaient encore obligatoires.
5. Le bias cible est **TPS65132WRVCT (C1848345, WQFN-20 3 × 4 mm)** ; le TPS65132A0YFFR OEM Moto reste une référence, mais ne doit pas être choisi s’il force Standard PCBA.
6. Conserver la cible Moto **+5,5 V / -5,7 V** ; programmer la NVM via les 4 pads avant validation finale de la dalle.
7. **Contrainte fabrication : rester en JLCPCB Economic PCBA.** Ne pas introduire de composant qui impose Standard sans nouvelle décision explicite.
8. Le backlight de premier essai est le **SY7203 Q6A**, avec R315 retirée, RSET initial ~8,2 Ω, 3×47 Ω et mesure différentielle des trois branches.
9. Ne jamais peupler plusieurs straps RSET simultanément sauf calcul volontaire explicite.
10. Vérifier physiquement C644 avant modification : le PDF indique 10 V mais le composant monté peut être différent.
11. Le tactile conserve les **74LVC2T45 directionnels** ; ne pas réintroduire TXS/TXB par défaut.
12. Demander les rapports finaux touch/display seulement si un détail logiciel est nécessaire.
13. Demander les schémas Radxa/Moto pour toute décision de pinout/électrique.
14. Demander `V1 (2)(1).zip` uniquement pour récupérer l’ancien KiCad et les bibliothèques.
15. Ne pas recompiler tout Android : le besoin identifié est kernel touch + DTBO/vendor ciblés.
16. Tout nouveau schéma interposer doit partir de l’architecture **cible** section 12.
17. Ne pas figer un écran aftermarket comme OEM avant lecture/mesure réelle.
18. Ne jamais appliquer le firmware tactile xpeng avant validation du trim ID.
19. Pour fabrication : revalider MPN, footprints, orientation/connecteurs, stock JLC, ERC, SI DSI, rails et thermique.
20. Le modem cible est désormais **Quectel EC25-EUX sur core board CC-MCore-EC25EUXGR** ; ne pas réintroduire le SIMCom A7672E sans nouvelle décision explicite.
21. Ne pas traiter la core board comme un EC25 nu : distinguer capacité du module, routage de la board et capacité du firmware exact.
22. En mode Android, la liaison principale EC25↔Q6A est **USB2 + QMI/RmNet** avec **Quectel RIL AIDL3 V4.6 / Radio AIDL V3** ; ne pas réadapter QCRIL Qualcomm au modem externe.
23. Pour le kernel EC25, activer `USB_SERIAL_WWAN`, `USB_SERIAL_OPTION`, `USB_WDM`, `USB_NET_QMI_WWAN` en built-in ; ne pas injecter les drivers Quectel externes tant que le kernel natif suffit.
24. Retirer `noril` et neutraliser seulement les services/manifests Radio/IMS Qualcomm qui concurrencent la pile EC25 ; conserver les fonctions Q6A non liées au modem externe.
25. La VoLTE reste **optionnelle et conditionnelle** : firmware + MBN + SIM + opérateur. Ne pas la déclarer acquise avant `QCFG ims`, `QMBNCFG list` et test réel.
26. Tester UAC sur le firmware exact avant de le déclarer acquis. Sauvegarder `USBCFG` avant toute modification.
27. En Minimal Phone, le Q6A doit pouvoir être totalement hors tension ; le MCU contrôle l'EC25 par UART, surveille **RI**, pilote **DTR**, et la voix passe par **PCM + codec dédié** si UAC/Q6A ne sont pas disponibles.
28. La cible d'autonomie est **EC25 alimenté + enregistré + Sleep (`QSCLK`)**, MCU always-on, Q6A suspendu ou OFF. `RI -> MCU -> wake Q6A` est le chemin de réveil privilégié à valider.
29. `RI` et `DTR` sont physiquement accessibles sur la core board ; leurs niveaux électriques réels doivent néanmoins être mesurés avant connexion GPIO finale.
30. `PEN` est un signal de la core board dont la fonction exacte reste à mesurer. Ne pas supposer `PEN=PWRKEY`. Si PEN coupe réellement le rail, l'utiliser après arrêt propre pour zéro consommation/recovery.
31. Une coupure brute d'alimentation EC25 est acceptable en recovery, pas comme shutdown normal. Préférer `AT+QPOWD`/PWRKEY puis couper le rail.
32. Les pads périphériques du EC25 sont accessibles ; récupérer directement **PCM 24–27 + I²C 41–42** uniquement si nécessaire. Respecter strictement le domaine 1,8 V.
33. Ne pas reprendre USB D+/D− par fils si le connecteur de la core board transporte déjà l'USB natif : éviter les stubs sur le lien High-Speed.
34. Lorsque le Q6A est OFF, vérifier `USB_VBUS` et tout risque de back-power avant de figer le schéma Minimal Phone.
35. Le MCU est le superviseur matériel : tous les boutons physiques arrivent au MCU, il contrôle PWR et la coupure d'alimentation Q6A.
36. Pour les boutons Android, privilégier **MCU I²C esclave + MCU_IRQ + driver Linux input** ; ne pas les gérer par polling dans une application Android.
37. Réserver si possible deux GPIO de secours pour Volume +/- direct afin de ne pas fermer la porte au bootloader/recovery.
38. L'extinction Q6A doit être coopérative : shutdown propre puis signal SAFE_TO_CUT puis coupure physique. Hard cut seulement en secours.
39. Ne pas ajouter de DAC externe pour le haut-parleur Android : utiliser J18 EAR/AUX du Q6A et un ampli adapté.
40. Le jack J17 reste le jack Android natif ; son fonctionnement en Minimal Phone n'est pas une exigence V1.
41. ALC5616 est seulement le premier candidat de codec Minimal grâce à la documentation Quectel ; ne pas le figer avant validation BOM/consommation/firmware.
42. La supervision thermique de coque par le MCU est **une proposition seulement** : sonde externe -> MCU -> état thermique vers Linux/Android -> mitigation. Ne pas considérer les seuils, la sonde ni l’intégration kernel comme figés avant mesures sur prototype.

**Prochaines tâches d’ingénierie logiques : (1) valider la core board EC25 sur PC Linux — USB, firmware, Sleep/RI/DTR, IMS/MBN, SMS/appels/QMI/UAC — avant toute modification Android ; (2) préparer le patch/build Q6A EC25 AIDL3 sans flash ; (3) redessiner l’interposer écran ; (4) figer la supervision MCU/Q6A/EC25, le contrôle USB_VBUS/power et l’audio Minimal, puis ERC/layout en restant compatible JLCPCB Economic PCBA.**

---

# 27. Addendum — décisions et informations ajoutées le 2026-09-22

Cet addendum résume les éléments nouveaux intégrés depuis le gel du 2026-09-16. Il **ne modifie pas le périmètre physique figé de la PCB écran**.

```text
Android stock : SHA-256 local ajouté ; eMMC réelle YMTC EC150 64 GB identifiée ; boot HDMI stock + inventaire telephony à faire avant patch
Touch          : stratégie SPI directe conservée ; pont RP2040 -> USB HID = prototype alternatif seulement
Display        : Redmi Note 7 étudié comme alternative potentiellement plus simple ; non adopté
EC25           : marquage commercial EC25EUXGR-128-SGNS observé sur photo vendeur ; à confirmer sur module reçu
RF EC25        : MAIN=LTE principal, AUX=diversité LTE, GNSS=satellites
VoLTE          : validation finale exige appel IMS/LTE réel + audio duplex sur SIM/opérateur cible
```

Les manipulations Termux/Claude Code réalisées pendant la phase d’outillage **ne constituent pas une décision d’architecture Maker Phone** et ne sont pas intégrées comme dépendance du produit.

---

# 28. Inspection physique du matériel reçu — 2026-09-22

Cette section consigne uniquement ce qui est visible sur les exemplaires réellement reçus. Elle prime sur les hypothèses fondées uniquement sur les photos vendeur.

## 28.1 Radxa Dragon Q6A

[CONFIRMÉ PAR PHOTO]

```text
PCB réelle : Dragon Q6A V1.21
SoC visible : Qualcomm QCS6490
état visuel : aucun dommage évident observé sur les photos
mise sous tension : pas encore effectuée au moment de cette inspection
```

La révision sérigraphiée `V1.21` correspond à la révision des schémas et du placement map déjà utilisés dans le projet. Cela réduit le risque de divergence matérielle entre la carte reçue et la documentation de référence, sans remplacer les mesures électriques.

## 28.2 Quectel EC25 et core board

[CONFIRMÉ PAR PHOTO]

Marquage du module :

```text
EC25-EUX
GA
EC25EUXGA-128-SGNS
```

La mention antérieure `GR / EC25EUXGR-128-SGNS`, issue d’une photo commerciale, est donc invalidée pour l’exemplaire reçu.

Breakouts visibles :

```text
rangée 1 :
VIN
GND
PWK
RST
PCM IN
PCM OUT
PCM SYNC
PCM CLK
SDA
SCL
AD0
AD1

rangée 2 :
VBUS
DN
DP
NET
GND
RXD
TXD
VIO
DTR
RI
BAT
GND
```

Éléments visibles supplémentaires :

```text
slot SIM
slot TF/microSD
micro-USB
3 connecteurs RF : DIV / GNS / MAIN
```

Conséquence importante : **PCM + I²C semblent déjà ramenés sur le breakout de la core board**. Le rework direct sur les pads LCC 24–27 / 41–42 ne doit donc plus être considéré comme la voie normale tant que ce breakout n’a pas été testé.

[À VALIDER AVANT CONNEXION MCU/CODEC]

```text
continuité réelle des broches PCM/I²C vers le EC25
niveau logique de VIO
niveau logique UART
niveau logique PCM/I²C
polarité/niveau PWK, RST, DTR, RI
fonction exacte NET
fonction exacte PEN/enable si présente ailleurs sur cette révision
comportement alimentation VIN/BAT
```

## 28.3 Écran reçu

[CONFIRMÉ PAR PHOTO]

Marquages visibles :

```text
FMS780 JB2-1
QL1-12
FPC principal marqué 1 .. 40
```

La face arrière ne présente pas, sur les photos disponibles, de marquage explicite `Tianma` ou `NT36672E`. L’identité du contrôleur/panel supplier reste donc [À VALIDER].

Le petit flex de backlight porte visiblement :

```text
K3 K2 K1 A
```

Cela est cohérent avec la topologie déjà retenue :

```text
A  = anode commune
K1 = cathode string 1
K2 = cathode string 2
K3 = cathode string 3
```

Cette observation renforce la nécessité de mesurer séparément les trois courants de strings lors du premier essai avec le SY7203 + ballast 47 Ω.

## 28.4 eMMC

[CONFIRMÉ PAR PHOTO — 2026-09-22]

Module reçu :

```text
PCB : eMMC Module V1.2
fabricant eMMC : YMTC
marquage : YMEC7C0TG1A2C3
famille : YMTC EC150
capacité : 64 GB
interface : eMMC 5.1
```

Le marquage `YMEC7C0TG1A2C3` correspond à la variante 64 GB de la famille YMTC EC150. Cela confirme matériellement la capacité 64 GB prévue pour la V1.

Inspection visuelle :

```text
2 connecteurs mezzanine présents
contacts visuellement propres
aucun composant arraché ou dommage mécanique évident sur les photos
```

La capacité a ensuite été confirmée par Firehose/EDL sur `Sdcc`, slot 0 :

```text
secteurs LUN0 : 122159104 × 512 octets
capacité brute exposée : 62 545 461 248 octets
                       ≈ 62,55 GB décimaux
                       ≈ 58,25 GiB
```

Cette capacité est cohérente avec une eMMC commercialisée comme **64 GB**. Après flash Android, la GPT contient 88 entrées. Les allocations les plus importantes observées sont :

```text
userdata : 37631,86 MiB  (~36,75 GiB)
rawdump  : 12488,00 MiB  (~12,20 GiB)
super    :  6144,00 MiB  ( 6,00 GiB)
logdump  :   512,00 MiB
```

La taille de `userdata` n'est donc pas la capacité totale de l'eMMC : une part importante du stockage est réservée aux partitions Android/Qualcomm, en particulier `rawdump` et `super`. Toute optimisation future de la GPT devra être faite **après** conservation de la baseline stock et validation complète du boot/ADB.

L'identité logique détaillée de l'eMMC (`EXT_CSD`, révision, life-time/pre-EOL, etc.) reste à capturer depuis Android/Linux avant qualification complète du composant.

## 28.5 Suite immédiate après premier boot Android

```text
FAIT  — inspection eMMC/Q6A et installation eMMC
FAIT  — accès Qualcomm EDL / Sahara / Firehose
FAIT  — double sauvegarde QSPI usine vérifiée par SHA-256
FAIT  — flash firmware QSPI Android 20260630-b1
FAIT  — flash eMMC Android 15 stock 20260630-b1
FAIT  — validation GPT post-flash
FAIT  — premier boot Android stock ; interface Android atteinte
À FAIRE — capturer ADB/logcat/getprop/build fingerprint et baseline stockage
À FAIRE — inventorier la pile téléphonie stock (packages/features/APN/Telecom)
À FAIRE — conserver la QSPI factory backup hors machine de développement
À FAIRE — seulement ensuite commencer la caractérisation EC25 sur PC Linux
```

---

# 29. Bring-up EDL / flash Android stock réussi — 2026-09-22

[CONFIRMÉ PAR LOGS UTILISATEUR + BOOT RÉEL]

Cette section fige la baseline matérielle/logicielle obtenue avant toute modification Maker Phone.

## 29.1 Hôte et accès EDL

Environnement de flash validé :

```text
hôte : Windows 11 x64
outil : edl-ng 1.6.0
pilote : Qualcomm HS-USB QDLoader 9008
mode Q6A : Qualcomm EDL -> Sahara -> Firehose
liaison validée : USB-C côté PC -> USB-A 3.1 OTG côté Q6A
```

Le QCS6490 a été détecté par Sahara comme produit `QCS_KODIAK`. Le loader `prog_firehose_ddr.elf` de l'archive Radxa a été chargé en RAM puis Firehose a permis l'accès à la QSPI et à l'eMMC.

## 29.2 Archive Android utilisée

```text
archive : Q6A-Android15-spi-emmc-boot-20260630-b1.7z
SHA-256 : 1978e216beadb8bd9025bcab52814d1f0e16d01cbfee2fbd91c4795bd50ebb16
```

L'empreinte a été calculée localement avant extraction et correspond à la baseline déjà documentée en section 3.

L'archive contient deux cibles distinctes :

```text
spinor/     -> firmware boot QSPI associé à Android
emmc/asic/  -> image Android + firmware/partitions à écrire sur eMMC
```

## 29.3 Sauvegarde QSPI usine avant flash

Avant toute écriture, la QSPI a été lue via :

```text
memory : Spinor
capacité lue : 32,00 MiB
```

Deux lectures indépendantes ont produit exactement la même empreinte :

```text
Q6A_FACTORY_SPINOR_2026-09-22.bin
Q6A_FACTORY_SPINOR_2026-09-22_CHECK.bin

SHA-256 des deux fichiers :
0362790ce9310a742fc7952a8a8565e0e50f10fe7022df7ccb3783752fb62de9

taille : 33 554 432 octets = 32 MiB
```

Cette image brute est la référence de rollback de la QSPI **telle que reçue**. Elle doit être conservée en au moins une copie hors du PC de développement. Elle n'inclut ni l'eMMC ni les eFuses/QFPROM du SoC.

La GPT QSPI usine était lisible avant flash et contenait notamment `XBL`, `XBL_CONFIG`, `AOP`, `TZ`, `DEVCFG`, `HYP`, `QUP`, `CPUCP`, `SHRM`, `ImageFv`, `PILFv`, `VarStore`, etc.

## 29.4 Flash du firmware QSPI Android

Commande de principe validée :

```text
edl-ng --memory=spinor --loader prog_firehose_ddr.elf \
  rawprogram rawprogram0.xml patch0.xml
```

Résultat observé :

```text
Writing XBL ... 100 %
Writing PrimaryGPT ... 100 %
Writing BackupGPT ... 100 %
Patching LUN 0 ...
'rawprogram' command finished successfully.
```

Le firmware de boot QSPI correspondant à la release Android `20260630-b1` a donc été installé avec succès.

## 29.5 Détection eMMC avant flash

L'eMMC a été adressée par Firehose avec :

```text
memory : Sdcc
slot   : 0
```

Avant flash Android, le LUN0 était lisible et présentait une GPT simple avec une partition `primary` d'environ `59646 MiB`.

Les lectures GPT des LUN1/LUN2 ont renvoyé `No GPT` et le LUN3 un `NAK`. Ces messages ne bloquent pas le LUN0 et sont cohérents avec des zones matérielles eMMC qui ne portent pas une GPT utilisateur standard.

## 29.6 Flash Android eMMC

Le fichier `rawprogram_unsparse0.xml` a été inspecté avant écriture : toutes les opérations ciblent `physical_partition_number="0"`. `patch0.xml` cible également uniquement la partition physique 0 et ajuste la GPT à `NUM_DISK_SECTORS`.

Commande de principe validée :

```text
edl-ng --memory=Sdcc --slot 0 --loader prog_firehose_ddr.elf \
  rawprogram rawprogram_unsparse0.xml patch0.xml
```

Les écritures ont terminé à 100 %, notamment :

```text
xbl_a / xbl_b
xbl_config_a / xbl_config_b
boot_a / boot_b
super (segments super_1 ... super_6)
vbmeta_system_a
vendor_boot_a
modem_a / modem_b
dsp_a / dsp_b
abl_a / abl_b
bluetooth_a / bluetooth_b
dtbo_a / dtbo_b
persist
vbmeta_a / vbmeta_b
userdata (segments userdata_1 ... userdata_11)
PrimaryGPT / BackupGPT
```

Fin de procédure :

```text
Patching LUN 0 using patch0.xml
'rawprogram' command finished successfully.
```

## 29.7 Vérification GPT post-flash

Une lecture `printgpt` immédiatement après flash a confirmé une GPT Android valide de **88 entrées**.

Partitions structurantes observées :

```text
xbl_a / xbl_b
xbl_config_a / xbl_config_b
tz_a / tz_b
aop_a / aop_b
hyp_a / hyp_b
boot_a / boot_b
super                  6144,00 MiB
vbmeta_system_a/b
vendor_boot_a/b
metadata
keymaster_a/b
modem_a / modem_b
dsp_a / dsp_b
abl_a / abl_b
dtbo_a / dtbo_b
persist
logdump                  512,00 MiB
vbmeta_a / vbmeta_b
modemst1 / modemst2
fsg / fsc
rawdump                12488,00 MiB
userdata               37631,86 MiB
```

Géométrie eMMC LUN0 observée :

```text
Backup LBA       : 122159103
Last Usable LBA  : 122159070
secteur logique  : 512 octets
capacité exposée : 122159104 × 512
                  = 62 545 461 248 octets
                  ≈ 62,55 GB
                  ≈ 58,25 GiB
```

Le fait que `userdata` fasse environ 36,75 GiB ne signifie donc pas que l'eMMC a perdu de la capacité. Le layout stock réserve environ 12,20 GiB à `rawdump`, 6 GiB à `super`, 512 MiB à `logdump`, plus les nombreuses partitions firmware/Android A/B et des zones d'alignement.

## 29.8 Premier boot Android

[VALIDÉ — 2026-09-22]

Après coupure complète de l'alimentation, sortie du mode EDL et démarrage normal, **Android a démarré avec succès et l'utilisateur a atteint l'interface Android**.

État de qualification à ce jalon :

```text
Q6A V1.21 réel                  PASS
eMMC YMTC 64 GB réelle          PASS détection / flash / boot
EDL Qualcomm 9008               PASS
Sahara                           PASS
Firehose QSPI                    PASS
Firehose eMMC                    PASS
backup QSPI usine                PASS + double lecture SHA-256
flash firmware QSPI stock        PASS
flash Android 15 stock           PASS
GPT Android post-flash           PASS
premier boot Android stock       PASS
ADB / logs baseline              À CAPTURER
inventaire téléphonie stock      À CAPTURER
EC25 sur Q6A                     NON TESTÉ À CE JALON
écran Moto sur DSI Q6A           NON TESTÉ À CE JALON
```

## 29.9 Règle de baseline à partir de maintenant

Avant toute modification kernel, DTBO, modem, tactile, écran ou Android :

```text
1. capturer `adb devices`, `adb shell getprop`, build fingerprint et niveau de patch
2. capturer `df -h`, `/dev/block/by-name`, slots A/B et état AVB
3. conserver un `logcat -b all -d` et, si possible, `dmesg`
4. inventorier packages/features téléphonie stock
5. archiver hors machine le dump QSPI usine + son SHA-256
6. ne modifier la GPT (notamment rawdump/userdata) qu'après cette baseline
```

Cette baseline est désormais la référence de comparaison pour tous les futurs travaux Maker Phone.
