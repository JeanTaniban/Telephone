# NOTE 2026-10-03 — baseline logiciel conservée, hardware partiellement supersédé

> Les preuves Android/RIL/AT/SIM de ce document restent la baseline modem.  
> Pour le **hardware final EC25**, utiliser `ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-10-03_POWER_UPDATE.md` et `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md`.

Les points suivants de ce document sont historiques et ne doivent plus être recopiés dans le schéma V0.5 :

```text
- alimentation principale EC25 par VBUS USB Q6A ;
- absence de MODEM_PWR ;
- EC25 RESET_N consommant une GPIO MCU dédiée.
```

Architecture hardware courante :

```text
SYS -> MODEM_PWR -> carrier EC25 BAT
Q6A USB2 host -> carrier VBUS/DP/DN/GND
RESET_N -> option open-drain DNP depuis GP28 réserve
```

---

# EC25-EUXGA sur Q6A — état vérifié le 25 septembre 2026, architecture mise à jour le 30 septembre 2026

## Conclusion

Le modem **GA** communique avec Android 15 jusqu'au constat de **SIM absente**. Le EC25 répond aux commandes AT ; le RIL Quectel AIDL1 publie sept services Radio ; le framework Android reçoit `CARDSTATE_ABSENT` et expose `gsm.sim.state=ABSENT`. La chaîne complète a été observée sur le Q6A réel le 24 septembre. Le 25 septembre, la présence des deux processus, des sept services et de l'état `ABSENT` a été recontrôlée ; les échanges AT et les événements `RILJ` n'ont pas été rejoués ce jour-là.

L'installation est encore un **prototype de développement volontairement manuel**. Les changements de fichiers vivent dans des overlays `vendor`, `system_ext` et `product` ; le pont USB AT et le RIL sont lancés manuellement dans `/data/local/tmp` sous ADB root. Cette méthode est conservée pour avancer rapidement tant que le fonctionnement modem n'est pas suffisamment validé. Une fois validé, les changements devront être intégrés proprement aux images Android puis flashés de manière ciblée et reproductible.

Le lien matériel final Q6A ↔ EC25 est **USB 2.0 host depuis l'USB-A du Q6A** : `VBUS / D+ / D- / GND -> carrier EC25`. Le port USB-C d'alimentation du Q6A n'est pas le chemin modem. Pour l'alimentation principale finale du modem, voir la note 2026-10-03 ci-dessus.

## Ce qui est prouvé, et sa limite

| Point | Preuve sur la carte | Limite actuelle |
|---|---|---|
| Identité du modem | `AT+QGMR` → `EC25EUXGAR08A05M1G_01.001.01.001` ; USB `2c7c:0125`. | Les documents ou réglages GR ne sont pas présumés applicables au GA. |
| Accès AT | Interface USB 2, endpoints bulk OUT `0x03` / IN `0x84` ; `AT+CPIN?` → `+CME ERROR: 10`. | Accès assuré par un pont `usbfs`/PTY de développement, faute de pilote série `option`. |
| Radio Android | `service list` donne `IRadioConfig/default` et les six services `IRadio{Data,Messaging,Modem,Network,Sim,Voice}/slot1`. Les logs antérieurs montrent `RILJ`, `GET_SIM_STATUS` → `CARDSTATE_ABSENT`, puis `SIM_STATE_CHANGED ABSENT`. | Les services sont lancés par ADB root ; SELinux est `Enforcing` mais les processus tournent dans le domaine `su`, pas dans un domaine de production. |
| Slot et état SIM | `ro.telephony.sim_slots.count=1`, `telephony.active_modems.max_count=1`, `gsm.sim.state=ABSENT`. | Aucun test avec SIM insérée à la date de cette baseline. |
| Données cellulaires | Aucun `cdc-wdm*` ni `wwan*` du EC25 sur le noyau stock. | QMI/data, SMS, appels, audio, IMS/VoLTE et enregistrement réseau non validés. |
| Suspend-to-RAM | SHA-256 de `dtbo_a` toujours `25c95174e46f253ece0d5a5c9d0033d6d56a42fd9563d0edb5d2ed046baf6c42`. | Coexistence STR avec pilotes et modem intégrés à tester. |

## Relevé de contrôle du 25 septembre 2026

Depuis le PC, `adb devices` indiquait `8ba07537 device`. Sur le Q6a : `sys.boot_completed=1`, `getenforce=Enforcing`, `ro.boot.veritymode=disabled`, sept services Radio enregistrés, `gsm.sim.state=ABSENT`, et les processus `ec25_usb_pty` et `ec25_rild_probe` présents. `init.svc.vendor.qcrild=stopped`. Les montages `vendor`, `system_ext` et `product` montrent un système de fichiers ext4 en lecture seule sous un overlay utilisant `/mnt/scratch/overlay/...`. Le DTBO porte le SHA-256 ci-dessus.

Le contrôle a aussi renvoyé `persist.radio.multisim.config=ssss`, alors que le journal de l'essai précédent relevait `ss` et que le slot effectif reste `1`. **Cette divergence de propriété est à expliquer avant de figer l'image** ; elle ne remet pas en cause le résultat Radio/SIM observé. L'horloge du Q6a indiquait le 11 juin 2026 lors du relevé : les dates Android de fichiers ou de logs ne doivent pas servir seules à dater les essais ; la date du titre est celle du PC.

## Changements du prototype par partition

| Emplacement | Changement actuellement essayé | À reporter dans l'intégration durable |
|---|---|---|
| `vendor` | `ro.radio.noril=false`, `ro.vendor.radio.noril=false` ; exclusions téléphonie retirées ; manifeste VINTF Yupik converti de Radio HIDL à sept instances AIDL1 ; RIL Quectel et bibliothèques Radio V1 copiés sous `/vendor/lib64/ec25`. | Images vendor, service `init`, chemins des bibliothèques, règles `ueventd` et SELinux ; vérifier le comportement des autres paquets Qualcomm. |
| `system_ext` | Deux propriétés de slots/modems passées de `2` à `1`. | Image `system_ext_a`. |
| `product` | Deux overlays Qualcomm masqués : `TelephonyResCommon_Sys.apk` (injection QtiRIL) et `FrameworksResNonModem_Sys.apk` (voix/SMS/data désactivés). | Image `product_a` avec retrait ciblé et contrôle des ressources résultantes. |
| `boot` | Aucun changement. Le noyau stock a `CONFIG_USB_SERIAL_OPTION`, `CONFIG_USB_WDM` et `CONFIG_USB_NET_QMI_WWAN` désactivés. | Source Radxa exacte et toolchain compatible, puis activation des pilotes et vérification du mapping réel des interfaces USB GA. |
| `dtbo` | Aucun changement pendant les essais modem. Le patch STR préexistant est conservé. | Préserver l'image patchée ; vérifier son SHA avant/après tout flash. |

Les huit bibliothèques Radio V1 du paquet AIDL1 proviennent de l'APEX VNDK v33 présent sur cette image ; la copie isolée a résolu le chargement du RIL. Le paquet AIDL3 demanderait huit bibliothèques V3 absentes : **AIDL1 est la base du prototype validé**, pas une preuve que toutes les fonctions cellulaires sont opérationnelles.

---

# Mise à jour architecture — 30 septembre 2026

## SIM / réseau cible de qualification

La prochaine qualification réelle utilise une **SIM Orange avec PIN actif**. Une autre ligne/téléphone est disponible pour générer appels et SMS.

Avant commande de la PCB Power/MCU, le projet ne cherche plus à terminer toute l'intégration radio. Le minimum utile est :

```text
[ ] SIM Orange détectée
[ ] saisie/acceptation PIN
[ ] enregistrement réseau si possible
[ ] appel entrant et sortant au moins établi si la pile actuelle le permet
[ ] mesure électrique VIO/TXD/RI du carrier
```

Data Android, SMS complets, audio duplex, IMS/VoLTE, démarrage autonome du RIL et intégration image **restent souhaitables mais ne sont pas des blockers PCB**. Si ces fonctions échouent après commande, l'intégration sera adaptée logiciellement sans remettre en cause la liaison USB Q6A ↔ EC25 ni le level-shifter MCU.

## Audio — simplification V1

L'ancienne cible « Minimal Phone capable d'un appel vocal avec Q6A physiquement OFF via PCM EC25 + codec dédié » est **abandonnée pour V1**.

Nouvelle règle :

```text
TOUT AUDIO TELEPHONIE / MEDIA -> Q6A / Android
MCU -> aucun flux audio
EC25 PCM/I2C audio -> non requis V1
```

Le Q6A fournit la source analogique audio du téléphone via son codec. Architecture retenue au niveau système :

```text
Q6A HPH_L/R
  +--> oreillette interne toujours raccordée, sous réserve validation électrique
  +--> ampli stéréo haute impédance
          +--> HP gauche
          +--> HP droit

MCU -> coupe/alimente l'ampli HP via une sortie auxiliaire
micro interne -> entrée micro native Q6A à sélectionner/valider
Bluetooth -> audio externe
aucun jack utilisateur final
```

En lecture média, l'oreillette et les deux HP peuvent être actifs. Pendant un appel, le MCU coupe l'ampli des deux HP et l'oreillette reste le transducteur de sortie principal.

Avant fabrication, vérifier si possible que `HPH_L/R` peut être activé sans jack physiquement inséré et que l'oreillette + l'entrée haute impédance de l'ampli ne chargent pas excessivement le codec Q6A.

Conséquence modem : la réussite d'un appel ne dépend plus d'un codec Minimal branché sur PCM. Il faut en revanche toujours établir le chemin audio entre la téléphonie EC25/Android et le codec Q6A. Le mécanisme exact pourra être UAC/USB ou un autre routage supporté par la pile finale ; ce point reste logiciel/intégration et n'est pas un blocker de la carte Power/MCU.

## Minimal Phone — périmètre réduit

Le MCU conserve les fonctions de supervision basse consommation : boutons, réveil Q6A, RI/DTR/UART AT du EC25, état batterie/power et éventuellement petit écran arrière. En revanche, **un appel vocal nécessite désormais le réveil du Q6A**.

Le scénario cible devient :

```text
EC25 enregistré / événement entrant
        |
        +--> RI -> MCU
                |
                +--> MCU réveille Q6A
                        |
                        +--> Android prend en charge l'appel et l'audio
```

Le mode Minimal peut rester utile pour afficher des informations simples et superviser le modem, mais il n'essaie plus de reproduire un téléphone autonome complet avec Q6A éteint.

## Interface MCU ↔ EC25 conservée — historique 30/09

La simplification audio ne changeait alors pas :

```text
MCU UART TX/RX <-> EC25 RXD/TXD via SN74AVC4T245
MCU DTR        -> EC25 DTR via SN74AVC4T245
EC25 RI        -> MCU RI via SN74AVC4T245
MCU PWK/RST    -> open-collector S8050
```

Pour la V0.5 courante, `PWK` reste commandé ; `RST` est devenu une option DNP sur GP28 réserve, car `MODEM_PWR` fournit le dernier niveau de recovery.

L'UART permet l'envoi de commandes **AT** et la réception d'URC. Le niveau réel `VIO`, `TXD` et `RI` doit être mesuré sur le carrier avant connexion définitive.

---

## Prochaines étapes d'intégration complète après la commande PCB

1. **Qualification SIM/réseau** : poursuivre avec la SIM Orange réelle, PIN, réseau, appels, SMS/data selon disponibilité de la pile.
2. **Pilotes EC25** : obtenir le noyau Radxa exact `5.4.295-qgki-debug-g646f05065a7e-dirty` et son environnement de compilation ; intégrer `USB_SERIAL_WWAN`, `USB_SERIAL_OPTION`, `USB_WDM`, `USB_NET_QMI_WWAN`. Vérifier sur la carte les pilotes liés à chaque interface, les nœuds `ttyUSB*`/`cdc-wdm*`, l'interface réseau, puis les réponses AT.
3. **Démarrage autonome** : installer un `rild` compatible avec le RIL Quectel AIDL1, son service `init`, les droits de périphériques et la politique SELinux minimale. Après un reboot sans ADB root ni script manuel, les sept services doivent revenir.
4. **Images et flash ciblé** : reconstruire les partitions nécessaires puis flasher la nouvelle image ciblée avec AVB/verity cohérents, sans perdre le DTBO STR.
5. **Qualification fonctionnelle finale** : QMI/data, SMS, appels, audio duplex, puis STR/réveil avec modem branché.
6. **VoLTE/IMS Orange** : souhaitable si possible, mais à traiter après la chaîne téléphonie de base ; ne pas bloquer la PCB Power/MCU dessus.

Le [journal d'essai](notes/ec25_integration_2026-09-24.md) contient les observations chronologiques et le retour arrière ; le [plan d'intégration](PLAN_INTEGRATION_EC25_EUXGA_ANDROID15.md) détaille les dépendances par couche ; la [baseline GA](notes/ec25_euxga_baseline_2026-09-24.md) conserve les réponses AT et l'état initial. Le script [`scripts/start_ec25_probe.ps1`](scripts/start_ec25_probe.ps1) relance le prototype après reboot si ses binaires sont encore présents.
